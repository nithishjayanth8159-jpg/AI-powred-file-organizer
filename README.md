# AI-powred-file-organizer
#File organizer
from __future__ import annotations
 
import argparse
import calendar
import csv
import hashlib
import json
import logging
import os
import re
import shutil
import sys
import time
import unicodedata
from concurrent.futures import ThreadPoolExecutor
from dataclasses import dataclass
from datetime import datetime
from pathlib import Path
from typing import Iterable, Iterator, Optional
 
try:
    from pypdf import PdfReader
except ImportError:  # metadata features degrade gracefully
    PdfReader = None
try:
    from tqdm import tqdm
except ImportError:
    tqdm = None
 
log = logging.getLogger("pdf_organizer")
 
DEFAULT_CATEGORIES = {
    "Invoices & Receipts": ["invoice", "bill", "receipt", "gst", "purchase order"],
    "Bank & Statements": ["statement", "bank", "passbook", "transaction"],
    "Resumes": ["resume", "cv", "curriculum vitae"],
    "Contracts & Legal": ["contract", "agreement", "nda", "lease", "policy", "terms"],
    "Certificates": ["certificate", "certification", "diploma", "marksheet", "transcript"],
    "Reports": ["report", "annual", "summary", "analysis", "whitepaper"],
    "Manuals & Guides": ["manual", "guide", "handbook", "datasheet", "tutorial"],
    "Research Papers": ["paper", "journal", "arxiv", "proceedings", "abstract", "thesis"],
    "Study Material": ["notes", "lecture", "syllabus", "assignment", "exam",
                       "question paper", "chapter", "unit"],
    "Books": ["book", "ebook", "edition", "novel", "isbn"],
}
WINDOWS_RESERVED = {"CON", "PRN", "AUX", "NUL", *(f"COM{i}" for i in range(1, 10)),
                    *(f"LPT{i}" for i in range(1, 10))}
 
 
# ----------------------------------------------------------------------------
# Data
# ----------------------------------------------------------------------------
@dataclass
class PdfInfo:
    path: Path
    size: int
    mtime: float
    title: str = ""
    author: str = ""
    pages: int = 0
    created: Optional[datetime] = None
    status: str = "ok"  # ok | encrypted | corrupt
 
    def date(self, source: str) -> datetime:
        if source == "meta" and self.created:
            return self.created
        return datetime.fromtimestamp(self.mtime)
 
 
@dataclass
class Op:
    src: Path
    dst: Path
    note: str  # ok | duplicate | encrypted | corrupt
    info: Optional[PdfInfo] = None
 
 
# ----------------------------------------------------------------------------
# Helpers
# ----------------------------------------------------------------------------
def natural_key(text: str):
    return [int(t) if t.isdigit() else t for t in re.split(r"(\d+)", text.lower())]
 
 
def progress(iterable: Iterable, total: int, desc: str):
    if tqdm:
        return tqdm(iterable, total=total, desc=desc, unit="file")
    def gen():
        for i, x in enumerate(iterable, 1):
            if i % 500 == 0 or i == total:
                print(f"{desc}: {i}/{total}", file=sys.stderr)
            yield x
    return gen()
 
 
def scan_pdfs(root: Path, skip: Optional[Path]) -> Iterator[Path]:
    """Iterative, symlink-safe recursive scan (no recursion limit issues)."""
    stack = [root]
    skip_r = skip.resolve() if skip else None
    while stack:
        current = stack.pop()
        try:
            with os.scandir(current) as it:
                for entry in it:
                    try:
                        if entry.is_symlink():
                            continue
                        if entry.is_dir(follow_symlinks=False):
                            if skip_r and Path(entry.path).resolve() == skip_r:
                                continue
                            stack.append(Path(entry.path))
                        elif entry.is_file(follow_symlinks=False) and \
                                entry.name.lower().endswith(".pdf"):
                            yield Path(entry.path)
                    except OSError as e:
                        log.warning("Skipping %s: %s", entry.path, e)
        except OSError as e:
            log.warning("Cannot read folder %s: %s", current, e)
 
 
def sha256(path: Path, chunk: int = 1 << 20) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        while block := f.read(chunk):
            h.update(block)
    return h.hexdigest()
 
 
def safe_component(s: str, fallback: str = "Unknown") -> str:
    s = re.sub(r'[<>:"/\\|?*\x00-\x1f]', "", s).strip(" .")
    return s[:80] or fallback
 
 
def pdf_date(value) -> Optional[datetime]:
    if isinstance(value, datetime):
        return value.replace(tzinfo=None)
    return None
 
 
def extract_info(path: Path, use_metadata: bool) -> PdfInfo:
    try:
        st = path.stat()
    except OSError:
        return PdfInfo(path, 0, time.time(), status="corrupt")
    info = PdfInfo(path, st.st_size, st.st_mtime)
    try:
        with open(path, "rb") as f:
            head = f.read(1024)
        if st.st_size == 0 or b"%PDF" not in head:
            info.status = "corrupt"
            return info
    except OSError:
        info.status = "corrupt"
        return info
    if not (use_metadata and PdfReader):
        return info
    try:
        reader = PdfReader(str(path), strict=False)
        if reader.is_encrypted:
            try:
                if not reader.decrypt(""):
                    info.status = "encrypted"
                    return info
            except Exception:
                info.status = "encrypted"
                return info
        info.pages = len(reader.pages)
        meta = reader.metadata
        if meta:
            info.title = str(meta.title or "").strip()
            info.author = str(meta.author or "").strip()
            try:
                info.created = pdf_date(meta.creation_date)
            except Exception:
                pass
    except Exception as e:
        log.debug("Unreadable PDF %s: %s", path, e)
        info.status = "corrupt"
    return info
 
 
# ----------------------------------------------------------------------------
# Naming
# ----------------------------------------------------------------------------
def clean_stem(stem: str, style: str, ascii_only: bool) -> str:
    s = unicodedata.normalize("NFKD" if ascii_only else "NFKC", stem)
    if ascii_only:
        s = s.encode("ascii", "ignore").decode()
    s = re.sub(r'[<>:"/\\|?*\x00-\x1f]', " ", s)
    s = re.sub(r"(?i)^copy of\s+", "", s)
    s = re.sub(r"(?i)[\s_-]+\(?copy\)?(\s*\d+)?$", "", s)   # "report - Copy"
    s = re.sub(r"\s*\(\d{1,2}\)$", "", s)                   # "report (1)"
    s = s.replace("_", " ")
    s = re.sub(r"(?<!\d)\.|\.(?!\d)", " ", s)               # dots, keep 1.2
    s = re.sub(r"\s+", " ", s).strip(" -._")
    if style == "title":
        s = " ".join(w if (w.isupper() and 1 < len(w) <= 5) else w[:1].upper() + w[1:].lower()
                     for w in s.split(" "))
    elif style == "snake":
        s = re.sub(r"[\W]+", "_", s).strip("_").lower()
    elif style == "kebab":
        s = re.sub(r"[\W_]+", "-", s).strip("-").lower()
    return s
 
 
def good_title(t: str) -> bool:
    if len(t) < 4 or len(t) > 150:
        return False
    return not re.match(r"(?i)^(untitled|microsoft word|microsoft powerpoint|"
                        r"document\d*|scan|new document)", t) and not t.lower().endswith(
        (".doc", ".docx", ".indd", ".tex", ".ppt", ".pptx"))
 
 
def build_name(info: PdfInfo, a, seq: int, width: int) -> str:
    base = info.title if (a.use_title and good_title(info.title)) else info.path.stem
    base = clean_stem(base, a.style, a.ascii) or "document"
    sep = {"snake": "_", "kebab": "-"}.get(a.style, " - ")
    if a.date_prefix:
        base = f"{info.date(a.date_source):%Y-%m-%d}{sep}{base}"
    if a.number:
        base = f"{seq:0{width}d}{sep}{base}"
    base = base[: a.max_len].rstrip(" ._-") or "document"
    if base.upper() in WINDOWS_RESERVED:
        base += "_"
    return base + ".pdf"
 
 
# ----------------------------------------------------------------------------
# Folder levels
# ----------------------------------------------------------------------------
class Classifier:
    def __init__(self, categories: dict[str, list[str]]):
        self.rules = [(name, re.compile(r"\b(" + "|".join(map(re.escape, kws)) + r")\b", re.I))
                      for name, kws in categories.items() if kws]
 
    def category(self, info: PdfInfo) -> str:
        text = f"{info.path.stem.replace('_', ' ')} {info.title}"
        for name, rx in self.rules:
            if rx.search(text):
                return name
        return "Other"
 
 
def make_levels(a, clf: Classifier):
    def pages(i):
        p = i.pages
        if not p:
            return "Unknown pages"
        return "1-10 pages" if p <= 10 else "11-50 pages" if p <= 50 else \
            "51-200 pages" if p <= 200 else "200+ pages"
 
    def size(i):
        mb = i.size / 1048576
        return "Small (under 1 MB)" if mb < 1 else "Medium (1-20 MB)" if mb <= 20 else "Large (over 20 MB)"
 
    def letter(i):
        c = clean_stem(i.path.stem, "keep", True)[:1].upper()
        return c if c.isalpha() else "#"
 
    return {
        "year": lambda i: str(i.date(a.date_source).year),
        "month": lambda i: f"{i.date(a.date_source).month:02d} - "
                           f"{calendar.month_name[i.date(a.date_source).month]}",
        "category": clf.category,
        "author": lambda i: safe_component(i.author, "Unknown Author"),
        "pages": pages,
        "size": size,
        "letter": letter,
    }
 
 
# ----------------------------------------------------------------------------
# Planning
# ----------------------------------------------------------------------------
def find_duplicates(paths: list[Path], workers: int) -> dict[Path, Path]:
    """Returns {duplicate_path: original_path}. Only hashes same-size files."""
    by_size: dict[int, list[Path]] = {}
    for p in paths:
        try:
            by_size.setdefault(p.stat().st_size, []).append(p)
        except OSError:
            pass
    candidates = [p for size, g in by_size.items() if len(g) > 1 and size > 0 for p in g]
    if not candidates:
        return {}
    dupes: dict[Path, Path] = {}
    seen: dict[tuple[int, str], Path] = {}
    candidates.sort(key=lambda p: natural_key(str(p)))
    with ThreadPoolExecutor(workers) as ex:
        hashes = ex.map(lambda p: _safe_hash(p), candidates)
        for p, h in progress(zip(candidates, hashes), len(candidates), "Hashing"):
            if h is None:
                continue
            key = (p.stat().st_size, h)
            if key in seen:
                dupes[p] = seen[key]
            else:
                seen[key] = p
    return dupes
 
 
def _safe_hash(p: Path) -> Optional[str]:
    try:
        return sha256(p)
    except OSError as e:
        log.warning("Cannot hash %s: %s", p, e)
        return None
 
 
def unique_path(target: Path, taken: set[str]) -> Path:
    cand, n = target, 2
    while str(cand).lower() in taken or cand.exists():
        cand = target.with_name(f"{target.stem} ({n}){target.suffix}")
        n += 1
    taken.add(str(cand).lower())
    return cand
 
 
def sort_key(info: PdfInfo, a):
    if a.sort_by == "date":
        return (info.date(a.date_source), natural_key(info.path.name))
    if a.sort_by == "size":
        return (info.size, natural_key(info.path.name))
    if a.sort_by == "pages":
        return (info.pages, natural_key(info.path.name))
    return natural_key(info.path.name)
 
 
def plan(a, src: Path, dest: Path) -> list[Op]:
    paths = list(scan_pdfs(src, dest))
    log.info("Found %d PDF files", len(paths))
    if not paths:
        return []
 
    dupes = find_duplicates(paths, a.workers) if a.dupes != "ignore" else {}
    log.info("Exact duplicates: %d", len(dupes))
 
    unique = [p for p in paths if p not in dupes]
    infos: list[PdfInfo] = []
    with ThreadPoolExecutor(a.workers) as ex:
        for chunk_start in range(0, len(unique), 1000):  # bounded memory
            chunk = unique[chunk_start:chunk_start + 1000]
            infos.extend(progress(ex.map(lambda p: extract_info(p, not a.no_metadata), chunk),
                                  len(chunk), "Reading"))
    infos.sort(key=lambda i: sort_key(i, a))
 
    categories = dict(DEFAULT_CATEGORIES)
    if a.rules:
        categories = json.loads(Path(a.rules).read_text(encoding="utf-8"))
    clf = Classifier(categories)
    all_levels = make_levels(a, clf)
    levels = [l.strip() for l in a.by.split(",") if l.strip() and l.strip() != "none"]
    for l in levels:
        if l not in all_levels:
            sys.exit(f"Unknown --by level '{l}'. Choose from: {', '.join(all_levels)}")
 
    width = max(4, len(str(len(infos))))
    taken: set[str] = set()
    ops: list[Op] = []
    for seq, info in enumerate(infos, 1):
        name = build_name(info, a, seq, width)
        if info.status != "ok":
            folder = dest / "_Problem" / ("Encrypted" if info.status == "encrypted" else "Corrupt")
        else:
            folder = dest.joinpath(*(safe_component(all_levels[l](info)) for l in levels))
        ops.append(Op(info.path, unique_path(folder / name, taken), info.status, info))
 
    for dup, orig in sorted(dupes.items(), key=lambda kv: natural_key(str(kv[0]))):
        if a.dupes == "move":
            folder = dest / "_Duplicates"
            clean = clean_stem(dup.stem, a.style, a.ascii) or "document"
            ops.append(Op(dup, unique_path(folder / f"{clean}.pdf", taken), "duplicate"))
        else:  # delete
            ops.append(Op(dup, dup, "duplicate-delete"))
    return ops
 
 
# ----------------------------------------------------------------------------
# Execution / reporting / undo
# ----------------------------------------------------------------------------
def write_report(ops: list[Op], path: Path):
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "w", newline="", encoding="utf-8-sig") as f:
        w = csv.writer(f)
        w.writerow(["original", "new_path", "status", "pages", "size_kb", "author", "title"])
        for o in ops:
            i = o.info
            w.writerow([o.src, o.dst, o.note, i.pages if i else "", round(i.size / 1024) if i else "",
                        i.author if i else "", i.title if i else ""])
 
 
def execute(ops: list[Op], copy: bool, journal_path: Path):
    journal_path.parent.mkdir(parents=True, exist_ok=True)
    done, errors = [], 0
    action = "copy" if copy else "move"
    try:
        for op in progress(ops, len(ops), "Applying"):
            try:
                if op.note == "duplicate-delete":
                    op.src.unlink()
                    done.append({"src": str(op.src), "dst": None, "action": "delete"})
                    continue
                if op.src == op.dst:
                    continue
                op.dst.parent.mkdir(parents=True, exist_ok=True)
                (shutil.copy2 if copy else shutil.move)(str(op.src), str(op.dst))
                done.append({"src": str(op.src), "dst": str(op.dst), "action": action})
            except Exception as e:
                errors += 1
                log.error("Failed %s -> %s: %s", op.src, op.dst, e)
    finally:  # journal is always written, even on Ctrl+C
        journal_path.write_text(json.dumps(done, indent=1), encoding="utf-8")
    log.info("Done: %d operations, %d errors. Undo journal: %s", len(done), errors, journal_path)
 
 
def undo(journal: Path):
    entries = json.loads(journal.read_text(encoding="utf-8"))
    restored = skipped = 0
    for e in reversed(entries):
        src, dst = Path(e["src"]), Path(e["dst"]) if e["dst"] else None
        if e["action"] == "delete":
            log.warning("Cannot restore deleted file %s", src)
            skipped += 1
        elif e["action"] == "copy":
            if dst and dst.exists():
                dst.unlink()
                restored += 1
        elif dst and dst.exists() and not src.exists():
            src.parent.mkdir(parents=True, exist_ok=True)
            shutil.move(str(dst), str(src))
            restored += 1
        else:
            skipped += 1
    log.info("Undo complete: %d restored, %d skipped", restored, skipped)
 
 
def remove_empty_dirs(root: Path, keep: Path):
    for d, _, _ in os.walk(root, topdown=False):
        p = Path(d)
        if p == root or keep in p.parents or p == keep:
            continue
        try:
            p.rmdir()  # only succeeds if empty
        except OSError:
            pass
 
 
# ----------------------------------------------------------------------------
# CLI
# ----------------------------------------------------------------------------
def build_parser():
    p = argparse.ArgumentParser(description="Sort, rename & organize PDF files.")
    sub = p.add_subparsers(dest="cmd", required=True)
 
    o = sub.add_parser("organize", help="organize a folder of PDFs (dry-run unless --apply)")
    o.add_argument("source", type=Path)
    o.add_argument("--dest", type=Path, help="output folder (default: <source>/Organized)")
    o.add_argument("--apply", action="store_true", help="actually move/copy files")
    o.add_argument("--copy", action="store_true", help="copy instead of move")
    o.add_argument("--by", default="year",
                   help="comma list of folder levels: year,month,category,author,pages,size,letter,none")
    o.add_argument("--style", choices=["title", "snake", "kebab", "keep"], default="title",
                   help="filename style (default: title)")
    o.add_argument("--use-title", action="store_true", help="name files by PDF metadata title when good")
    o.add_argument("--date-prefix", action="store_true", help="prefix names with YYYY-MM-DD")
    o.add_argument("--date-source", choices=["meta", "mtime"], default="meta",
                   help="date from PDF metadata (falls back to modified time) or file mtime")
    o.add_argument("--number", action="store_true", help="prefix sequence numbers in sorted order")
    o.add_argument("--sort-by", choices=["name", "date", "size", "pages"], default="name")
    o.add_argument("--max-len", type=int, default=100, help="max filename length (default 100)")
    o.add_argument("--ascii", action="store_true", help="convert names to plain ASCII")
    o.add_argument("--dupes", choices=["move", "delete", "ignore"], default="move",
                   help="exact duplicate handling (default: move to _Duplicates)")
    o.add_argument("--rules", type=Path, help='JSON file {"Category": ["keyword", ...]}')
    o.add_argument("--no-metadata", action="store_true", help="skip reading PDF metadata (fastest)")
    o.add_argument("--remove-empty-dirs", action="store_true")
    o.add_argument("--workers", type=int, default=min(16, (os.cpu_count() or 4) * 2))
 
    u = sub.add_parser("undo", help="revert a previous run using its journal JSON")
    u.add_argument("journal", type=Path)
    p.add_argument("-v", "--verbose", action="store_true")
    return p
 
 
def main():
    a = build_parser().parse_args()
    logging.basicConfig(level=logging.DEBUG if a.verbose else logging.INFO,
                        format="%(levelname)s: %(message)s")
    if a.cmd == "undo":
        return undo(a.journal)
 
    src = a.source.resolve()
    if not src.is_dir():
        sys.exit(f"Not a folder: {src}")
    dest = (a.dest or src / "Organized").resolve()
    if PdfReader is None and not a.no_metadata:
        log.warning("pypdf not installed - metadata features disabled (pip install pypdf)")
 
    ops = plan(a, src, dest)
    if not ops:
        return log.info("No PDF files found.")
 
    stamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    report = dest / "_logs" / f"report_{stamp}.csv"
    write_report(ops, report)
    counts: dict[str, int] = {}
    for o in ops:
        counts[o.note] = counts.get(o.note, 0) + 1
    log.info("Plan: %s", ", ".join(f"{k}={v}" for k, v in sorted(counts.items())))
    log.info("Report: %s", report)
 
    if not a.apply:
        for o in ops[:15]:
            print(f"  {o.src.name}\n    -> {o.dst.relative_to(dest) if o.dst != o.src else '(delete)'}")
        if len(ops) > 15:
            print(f"  ... and {len(ops) - 15} more (see the CSV report)")
        return log.info("DRY RUN - nothing was changed. Re-run with --apply to proceed.")
 
    execute(ops, a.copy, dest / "_logs" / f"ops_{stamp}.json")
    if a.remove_empty_dirs and not a.copy:
        remove_empty_dirs(src, dest)
 
 
if __name__ == "__main__":
    main()
 
