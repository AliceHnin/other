import segyio
import numpy as np
import os

script_dir = "your file"
input_file  = os.path.join(script_dir, "file2.sgy")
output_file = os.path.join(script_dir, "file2.sgy")


print("Reading traces...")
all_traces  = []
all_headers = []
samples     = None

with segyio.open(input_file, "r", ignore_geometry=True) as f:
    samples = f.samples.copy()
    total   = f.tracecount

    fldr_offset  = 0
    prev_fldr    = None
    max_fldr_seen = 0
    file_count   = 0

    for i in range(total):
        h    = dict(f.header[i])
        fldr = h[segyio.TraceField.FieldRecord]

        # Detect FLDR reset (new file boundary)
        if prev_fldr is not None and fldr < prev_fldr:
            fldr_offset += max_fldr_seen
            file_count  += 1
            print(f"  File boundary at trace {i+1}: "
                  f"FLDR reset {prev_fldr}→{fldr}, "
                  f"offset now {fldr_offset}")

        # Apply offset to make FLDR continuous
        new_fldr = fldr + fldr_offset
        h[segyio.TraceField.FieldRecord] = new_fldr

        max_fldr_seen = max(max_fldr_seen, fldr) if fldr_offset == (fldr_offset - fldr_offset % 1500) * 1 else fldr

        # Track rolling max per segment
        if prev_fldr is not None and fldr < prev_fldr:
            max_fldr_seen = fldr
        else:
            max_fldr_seen = max(max_fldr_seen, fldr)

        all_traces.append(f.trace[i].copy())
        all_headers.append(h)
        prev_fldr = fldr

        if (i + 1) % 50000 == 0:
            print(f"  Read {i+1} / {total}")

print(f"\nTotal files detected : {file_count + 1}")
print(f"Total traces read    : {len(all_traces)}")
print(f"Final FLDR max       : {all_headers[-1][segyio.TraceField.FieldRecord]}")

# ── Write output ──────────────────────────────────────────────
print("\nWriting renumbered file...")
spec            = segyio.spec()
spec.sorting    = None
spec.format     = 1
spec.samples    = samples
spec.tracecount = len(all_traces)

with segyio.create(output_file, spec) as out:
    out.bin.update(tsort=segyio.TraceSortingFormat.UNKNOWN_SORTING)
    for i in range(len(all_traces)):
        out.trace[i]  = all_traces[i]
        out.header[i] = all_headers[i]
        out.header[i].update({
            segyio.TraceField.TRACE_SEQUENCE_LINE: i + 1,
            segyio.TraceField.TRACE_SEQUENCE_FILE: i + 1,
        })
        if (i + 1) % 50000 == 0:
            print(f"  Written {i+1} / {len(all_traces)}")

print(f"\nDone! → {output_file}")

# ── Verify ────────────────────────────────────────────────────
print("\n--- VERIFICATION ---")
with segyio.open(output_file, "r", ignore_geometry=True) as f:
    total = f.tracecount
    first_fldr = f.header[0][segyio.TraceField.FieldRecord]
    last_fldr  = f.header[total-1][segyio.TraceField.FieldRecord]
    print(f"Total traces : {total:,}")
    print(f"FLDR first   : {first_fldr}")
    print(f"FLDR last    : {last_fldr}")
    print(f"Expected max : {last_fldr} shots × 24 channels = {last_fldr * 12 or 24:,} traces")
