import pdfplumber
import pandas as pd
import re
from pathlib import Path

PDF_FOLDER = r"C:\Users\Ritesh\Downloads\Payment Stub_NY\MD\AT\Data"
OUTPUT_FILE = r"C:\Users\Ritesh\Downloads\Payment Stub_NY\MD\AT\Data\Combined_Attendance.csv"

all_rows = []

for pdf_file in Path(PDF_FOLDER).glob("*.pdf"):

    print(f"Processing : {pdf_file.name}")

    pdf_name = pdf_file.stem.split()[0]

    try:

        with pdfplumber.open(pdf_file) as pdf:

            first_page_text = pdf.pages[0].extract_text()

            service_start = ""
            service_end = ""

            m = re.search(
                r'(\d{1,2}/\d{1,2}/\d{4})-(\d{1,2}/\d{1,2}/\d{4})',
                first_page_text
            )

            if m:
                service_start = pd.to_datetime(
                    m.group(1)
                ).strftime("%m-%d-%Y")

                service_end = pd.to_datetime(
                    m.group(2)
                ).strftime("%m-%d-%Y")

            for page in pdf.pages:

                tables = page.extract_tables()

                if not tables:
                    continue

                for table in tables:

                    for row in table:

                        if not row:
                            continue

                        row = [str(x).strip() if x else "" for x in row]

                        if len(row) < 4:
                            continue

                        child_name = row[0].replace("\n", " ").strip()
                        voucher = row[1].strip()
                        non_traditional = row[2].strip()

                        if not re.fullmatch(r"\d{7}", voucher):
                            continue

                        attendance = row[3]

                        lines = [
                            x.strip()
                            for x in attendance.split("\n")
                            if x.strip()
                        ]

                        week1 = lines[0] if len(lines) > 0 else ""
                        week2 = lines[1] if len(lines) > 1 else ""

                        def get_day(day):

                            p1 = re.search(
                                rf"{day}\[[X]?\]",
                                week1
                            )

                            p2 = re.search(
                                rf"{day}\[[X]?\]",
                                week2
                            )

                            v1 = p1.group(0) if p1 else f"{day}[]"
                            v2 = p2.group(0) if p2 else f"{day}[]"

                            return f"{v1}\n{v2}"

                        all_rows.append({
                            "PDF Name": pdf_name,
                            "Service Start Date": service_start,
                            "Service End Date": service_end,
                            "Child Name": child_name,
                            "Voucher Number": voucher,
                            "Non-Traditional Hours": non_traditional,
                            "M": get_day("M"),
                            "Tu": get_day("Tu"),
                            "W": get_day("W"),
                            "Th": get_day("Th"),
                            "F": get_day("F"),
                            "Sa": get_day("Sa"),
                            "Su": get_day("Su")
                        })

    except Exception as e:
        print(f"Error : {pdf_file.name}")
        print(str(e))

df = pd.DataFrame(all_rows)

column_order = [
    "PDF Name",
    "Service Start Date",
    "Service End Date",
    "Child Name",
    "Voucher Number",
    "Non-Traditional Hours",
    "M",
    "Tu",
    "W",
    "Th",
    "F",
    "Sa",
    "Su"
]

df = df[column_order]

df.to_csv(
    OUTPUT_FILE,
    index=False,
    encoding="utf-8-sig"
)

print("=" * 60)
print("DONE")
print("Records :", len(df))
print("Output :", OUTPUT_FILE)
print("=" * 60)
