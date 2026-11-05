# python-mini-project. prn- 26013030100 name-adesh jadhav
import argparse
import csv
import random
import re
import statistics
from collections import defaultdict
from dataclasses import dataclass
from datetime import date, datetime, timedelta
from difflib import SequenceMatcher
from pathlib import Path

FREQUENCIES = [
    ("weekly", 7, 2, 52, 3),
    ("monthly", 30, 4, 12, 3),
    ("quarterly", 91, 7, 4, 2),
    ("yearly", 365, 10, 1, 2),
]

NOISE_WORDS = {
    "COM", "WWW", "POS", "UPI", "PAYMENT", "TO", "PURCHASE", "DEBIT",
    "CARD", "INC", "LTD", "PVT", "ORDER", "AUTOPAY", "RECURRING",
}

PRICE_HIKE_THRESHOLD = 0.05


def parse_date(text):
    for fmt in ("%Y-%m-%d", "%d/%m/%Y", "%d-%m-%Y", "%m/%d/%Y", "%d %b %Y"):
        try:
            return datetime.strptime(text.strip(), fmt).date()
        except ValueError:
            continue
    raise ValueError(f"Unrecognised date format: {text!r}")


def load_transactions(path):
    rows = []
    with open(path, newline="", encoding="utf-8-sig") as f:
        for raw in csv.DictReader(f):
            row = {k.strip().lower(): (v or "").strip() for k, v in raw.items() if k}
            try:
                amount = float(row["amount"].replace(",", ""))
                rows.append((parse_date(row["date"]), row["description"], amount))
            except (KeyError, ValueError):
                continue
    if any(a < 0 for _, _, a in rows):
        rows = [(d, desc, -a) for d, desc, a in rows if a < 0]
    return [r for r in rows if r[2] > 0]


def normalize(description):
    cleaned = re.sub(r"[^A-Za-z ]+", " ", description.upper())
    words = [w for w in cleaned.split() if w not in NOISE_WORDS and len(w) > 1]
    return " ".join(words) or description.upper()


def group_by_merchant(txns, threshold=0.85):
    groups = defaultdict(list)
    for d, desc, amount in txns:
        key = normalize(desc)
        match = next(
            (k for k in groups if SequenceMatcher(None, k, key).ratio() >= threshold),
            None,
        )
        groups[match or key].append((d, amount))
    return groups


@dataclass
class Subscription:
    merchant: str
    frequency: str
    charges: int
    last_amount: float
    last_date: date
    next_date: date
    annual_cost: float
    active: bool
    hike_from: float = None


def classify_gap(median_gap):
    for name, days, tol, per_year, min_n in FREQUENCIES:
        if abs(median_gap - days) <= tol:
            return name, days, tol, per_year, min_n
    return None


def detect_subscriptions(groups, as_of):
    found = []
    for merchant, charges in groups.items():
        charges.sort()
        if len(charges) < 2:
            continue
        dates = [d for d, _ in charges]
        amounts = [a for _, a in charges]
        gaps = [(b - a).days for a, b in zip(dates, dates[1:])]

        freq = classify_gap(statistics.median(gaps))
        if not freq:
            continue
        name, nominal, tol, per_year, min_n = freq
        if len(charges) < min_n:
            continue
        if sum(abs(g - nominal) <= tol for g in gaps) / len(gaps) < 0.7:
            continue

        last_date, last_amount = dates[-1], amounts[-1]
        active = (as_of - last_date).days <= nominal * 1.5 + tol

        next_date = last_date + timedelta(days=nominal)
        while active and next_date < as_of:
            next_date += timedelta(days=nominal)

        hike_from = None
        earlier = amounts[:-1]
        if earlier:
            typical = statistics.median(earlier)
            if last_amount > typical * (1 + PRICE_HIKE_THRESHOLD):
                hike_from = typical

        found.append(
            Subscription(
                merchant, name, len(charges), last_amount, last_date,
                next_date, last_amount * per_year, active, hike_from,
            )
        )
    return sorted(found, key=lambda s: s.annual_cost, reverse=True)


def print_report(subs, as_of, window_days, cur):
    active = [s for s in subs if s.active]
    lapsed = [s for s in subs if not s.active]

    print(f"\n{'SUBSCRIPTION LEAK REPORT':^78}")
    print(f"{'as of ' + as_of.isoformat():^78}\n")
    header = f"{'Merchant':<22}{'Freq':<11}{'Last charge':>13}{'Per year':>13}  {'Next due':<11}"
    print(header)
    print("-" * len(header))
    for s in active:
        flag = "  ^ PRICE HIKE" if s.hike_from else ""
        print(
            f"{s.merchant[:21]:<22}{s.frequency:<11}"
            f"{cur + format(s.last_amount, ',.0f'):>13}"
            f"{cur + format(s.annual_cost, ',.0f'):>13}  "
            f"{s.next_date.isoformat():<11}{flag}"
        )

    total = sum(s.annual_cost for s in active)
    print("-" * len(header))
    print(f"{'TOTAL (active)':<46}{cur + format(total, ',.0f'):>13}  ~ {cur}{total / 12:,.0f}/month\n")

    hikes = [s for s in active if s.hike_from]
    if hikes:
        print("Price hikes detected:")
        for s in hikes:
            extra = (s.last_amount - s.hike_from) * (s.annual_cost / s.last_amount)
            print(f"  - {s.merchant}: {cur}{s.hike_from:,.0f} -> {cur}{s.last_amount:,.0f} (+{cur}{extra:,.0f}/year)")
        print()

    soon = [s for s in active if 0 <= (s.next_date - as_of).days <= window_days]
    print(f"Renewing in the next {window_days} days:")
    if soon:
        for s in sorted(soon, key=lambda x: x.next_date):
            print(f"  - {s.next_date}  {s.merchant}  ({cur}{s.last_amount:,.0f})")
    else:
        print("  (none)")

    if lapsed:
        print("\nLooks cancelled already: " + ", ".join(s.merchant for s in lapsed))
    print()


def export_ics(subs, path, cur):
    stamp = datetime.now().strftime("%Y%m%dT%H%M%S")
    lines = ["BEGIN:VCALENDAR", "VERSION:2.0", "PRODID:-//Subscription Leak Detector//EN"]
    for s in subs:
        if not s.active:
            continue
        lines += [
            "BEGIN:VEVENT",
            f"UID:{normalize(s.merchant).replace(' ', '')}-{s.next_date:%Y%m%d}@leakdetector",
            f"DTSTAMP:{stamp}",
            f"DTSTART;VALUE=DATE:{s.next_date:%Y%m%d}",
            f"SUMMARY:Renewal: {s.merchant} ({cur}{s.last_amount:,.0f})",
            "END:VEVENT",
        ]
    lines.append("END:VCALENDAR")
    path.write_text("\r\n".join(lines), encoding="utf-8")
    print(f"Calendar file written to {path}")


def add_months(d, n):
    year, month = divmod(d.month - 1 + n, 12)
    return date(d.year + year, month + 1, min(d.day, 28))


def make_demo_csv(path, today):
    rnd = random.Random(42)
    rows = []

    def noisy(name):
        return f"{name} {rnd.randint(1000, 9999)}"

    start = add_months(today, -13)
    for k in range(14):
        rows.append((add_months(start, k), noisy("NETFLIX.COM"), 499 if k < 10 else 649))
        rows.append((add_months(start, k) + timedelta(days=4), noisy("SPOTIFY"), 119))
        rows.append((add_months(start, k) + timedelta(days=9), "CULT FIT GYM MEMBERSHIP", 1500))
        rows.append((add_months(start, k) + timedelta(days=12), noisy("AIRTEL POSTPAID"), 599))
    for k in range(3):
        rows.append((add_months(today, -3 + k) - timedelta(days=6), noisy("MINDFULNESS APP"), 799))

    rows.append((today - timedelta(days=20), "GOOGLE ONE STORAGE", 1999))
    rows.append((today - timedelta(days=385), "GOOGLE ONE STORAGE", 1999))
    rows.append((today - timedelta(days=325), "AMAZON PRIME MEMBERSHIP", 1499))
    rows.append((today - timedelta(days=690), "AMAZON PRIME MEMBERSHIP", 1499))

    shops = [("SWIGGY", 150, 700), ("ZOMATO", 200, 800), ("UBER TRIP", 120, 600),
             ("BIGBASKET", 400, 2500), ("AMAZON PAY INDIA", 300, 5000), ("DMART", 500, 3000)]
    for _ in range(90):
        name, lo, hi = rnd.choice(shops)
        rows.append((today - timedelta(days=rnd.randint(1, 400)), noisy(name), rnd.randint(lo, hi)))

    rows = sorted(r for r in rows if r[0] <= today)
    with open(path, "w", newline="", encoding="utf-8") as f:
        w = csv.writer(f)
        w.writerow(["date", "description", "amount"])
        for d, desc, amt in rows:
            w.writerow([d.isoformat(), desc, f"-{amt:.2f}"])
    print(f"Demo statement written to {path} ({len(rows)} transactions)")


def main():
    ap = argparse.ArgumentParser(description="Find forgotten subscriptions in a bank statement.")
    ap.add_argument("csv", nargs="?")
    ap.add_argument("--demo", action="store_true")
    ap.add_argument("--days", type=int, default=7)
    ap.add_argument("--ics", type=Path)
    ap.add_argument("--currency", default="Rs ")
    args = ap.parse_args()

    today = date.today()
    if args.demo:
        path = Path("sample_statement.csv")
        make_demo_csv(path, today)
    elif args.csv:
        path = Path(args.csv)
    else:
        ap.error("give a CSV file or use --demo")

    txns = load_transactions(path)
    if not txns:
        raise SystemExit("No usable transactions found. CSV header must be: date,description,amount")

    as_of = today if args.demo else max(today, max(d for d, _, _ in txns))
    subs = detect_subscriptions(group_by_merchant(txns), as_of)
    print_report(subs, as_of, args.days, args.currency)
    if args.ics:
        export_ics(subs, args.ics, args.currency)


if __name__ == "__main__":
    main()
