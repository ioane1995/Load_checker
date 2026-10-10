
# Truck Loads Data integrity Checker program

Below is the documentation for the program which i made using claude AI. for safety reasons i can not post whole code here, because i use this program every for my work.

"""
load_checker.py
----------------
Unified three-way shipment verifier, merging what used to be two separate
scripts (claude.py + ollama_parser.py, and dispatch_compare4.py + pt_common.py)
into one program that checks every load against ALL THREE sources of truth:

    1. PDF Rate Confirmation  - what the broker actually agreed to pay/ship
    2. ProTransport           - what dispatch entered into the TMS
    3. Dispatch Dashboard     - what's actually scheduled/assigned on the truck

For each load in the ProTransport Load Search grid, this now does in ONE pass
(one open/close of the Load Edit window, not two):

    1. Reads the grid row (Load #, Freight Amount, Pickup/Delivery, Broker
       Load Number) - this alone is enough for the rate/date-vs-PDF check.
    2. Opens the Load Edit window once and, while it's open:
         a. reads Driver / Truck / Trailer / Customer Load ID / pickup-delivery
            (whichever of these didn't already come off the grid row), and
         b. switches to the Files tab and opens the attached rate-con PDF.
    3. Extracts rate/pickup/delivery/load-number/broker from the PDF using the
       Universal Broker Parsing Engine (pdf_ratecon_extractor.py), optionally
       double-checked by a local Ollama model (ollama_parser.py) for any field
       the regex engine flagged as low/no-confidence.
    4. Looks the load up on the Dispatch Dashboard (by truck row first, by
       Customer/Broker Load ID second) to get Driver/Truck/Trailer and the
       dashboard's own pickup/delivery dates.
    5. Cross-checks all three sources and prints ONE unified report per load:
         - Rate:              PDF vs ProTransport vs Dashboard ('Trip Amount') (3-way)
         - Pickup / Delivery: PDF             vs ProTransport vs Dashboard (3-way)
         - Driver/Truck/Trailer: ProTransport vs Dashboard             (2-way)
         - Load / Customer reference: PDF vs ProTransport vs Dashboard (3-way)
         - Broker name:       PDF             vs Dashboard               (2-way)
         - Stops:             PDF only - flags a load whose Rate Confirmation
           references more than one distinct pickup or delivery date, since
           ProTransport/Dashboard only ever track a single pickup + single
           delivery and an extra stop is easy to leave out of the load entry.
    6. Closes the Load Edit window and the PDF viewer, then moves on.

Three-way DATE rule (this is the behavior that was specifically requested):
    - ProTransport's date matches BOTH the rate-con date AND the dashboard
      date                                                -> CORRECT
    - ProTransport's date matches ONLY ONE of them                 -> FLAGGED
      (the report says which of the two it disagrees with)
    - ProTransport's date matches NEITHER of them                  -> MISTAKE
    - Only one other source was available at all (the other lookup failed/was
      skipped) -> falls back to a straight 2-way match/mismatch against
      whichever one source IS available, clearly labeled as such.

Setup (one time):
    pip install pywinauto psutil pdfplumber python-dateutil playwright
    playwright install chromium
    Optional OCR fallback for scanned PDFs: pip install pytesseract pillow
    Optional local-LLM double-check: install & run Ollama, `ollama pull llama3.1:8b`

Usage:
    python load_checker.py                       # every load, every check
    python load_checker.py 5                      # only the first 5 loads
    python load_checker.py --loads 123456,789012   # specific PT load numbers
    python load_checker.py --start-from 62424       # resume a full run, skipping everything before this load
    python load_checker.py --start-from 62424 20    # resume at 62424, then only check the next 20
    python load_checker.py --fields rate,pickup,delivery   # pick any subset of the 7 checks
    python load_checker.py --fields 1,3,4          # same as above, by menu number (see --help)
    python load_checker.py --fields stops          # just the multi-stop detection, nothing else
    python load_checker.py --mode ratecon          # shortcut: PDF vs ProTransport only
    python load_checker.py --mode assignment       # shortcut: driver/truck/trailer only
    python load_checker.py --mode dates            # shortcut: 3-way pickup/delivery + load ref
    python load_checker.py --mistakes-only         # only print loads with issues
    python load_checker.py --debug                 # verbose diagnostics
    python load_checker.py --use-ollama             # enable local-LLM double-check
    python load_checker.py --dump-pt 123456         # dump PT grid row + UI tree
    python load_checker.py --dump-grid 123456       # dump just the grid row cells
    python load_checker.py --dash-dump 60115125689  # save dashboard HTML after search
    python load_checker.py --dash-lookup 60115125689  # dashboard-only lookup
"""

