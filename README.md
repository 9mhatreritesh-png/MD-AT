# MD-AT
convert file MD AT
import pdfplumber
import pandas as pd
import re
from pathlib import Path

PDF_FOLDER = r"C:\Users\Ritesh\Downloads\Payment Stub_NY\MD\AT\Data"
OUTPUT_CSV = r"C:\Users\Ritesh\Downloads\Payment Stub_NY\MD\AT\Data\Combined_Attendance.csv"

all_rows = []

for pdf_file in Path(PDF_FOLDER).glob("*.pdf"):

    print(f"Processing: {pdf_file.name}")

    pdf_name = pdf_file.stem

    service_period = ""

    try:
        with pdfplumber.open(pdf_file) as pdf:

            pages_text = []

            for page in pdf.pages:
                txt = page.extract_text()
                if txt:
                    pages_text.append(txt)

            full_text = "\n".join(pages_text)

            # Service Period
            sp = re.search(
                r'Service Period and Year\s*([0-9/]+-[0-9/]+)',
                full_text,
                re.IGNORECASE
            )

            if sp:
                service_period = sp.group(1)

            lines = full_text.split("\n")

            i = 0

            while i < len(lines):

                line = lines[i].strip()

                match = re.search(
                    r'^(.*?)\s+(\d{7})\s+([YN])\s+(.*)$',
                    line
                )

                if match:

                    child_name = match.group(1).strip()
                    voucher = match.group(2).strip()
                    non_traditional = match.group(3).strip()

                    week1 = match.group(4).strip()

                    week2 = ""
                    if i + 1 < len(lines):
                        week2 = lines[i + 1].strip()

                    def extract_day(day):

                        p1 = re.search(rf'{day}\[[X]?\]', week1)
                        p2 = re.search(rf'{day}\[[X]?\]', week2)

                        v1 = p1.group(0) if p1 else f"{day}[]"
                        v2 = p2.group(0) if p2 else f"{day}[]"

                        return f"{v1}\n{v2}"

                    row = {
                        "pdf name": pdf_name,
                        "Service Period and Year": service_period,
                        "9. Names of Children": child_name,
                        "10. Voucher Number": voucher,
                        "11. Non-Traditional Hours (Y or N)": non_traditional,
                        "M": extract_day("M"),
                        "Tu": extract_day("Tu"),
                        "W": extract_day("W"),
                        "Th": extract_day("Th"),
                        "F": extract_day("F"),
                        "Sa": extract_day("Sa"),
                        "Su": extract_day("Su")
                    }

                    all_rows.append(row)

                    i += 2
                    continue

                i += 1

    except Exception as e:
        print(f"Error in {pdf_file.name}: {e}")

df = pd.DataFrame(all_rows)

df.to_csv(
    OUTPUT_CSV,
    index=False,
    encoding="utf-8-sig"
)

print("=" * 60)
print("DONE")
print("Total Records :", len(df))
print("Output File :", OUTPUT_CSV)
print("=" * 60)
