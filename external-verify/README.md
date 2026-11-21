# External stability verification (2026-11-21)

Five independent runs of the shipped reproducer on fresh VMs (fresh qcow2
overlay per run, sequential replay).  Every run hit the target hang.

Command per run (in the guest, as root):

    syz-execprog -procs=1 -threaded -repeat=1 \
        -executor ./syz-executor reproducer/repro.kernsight.prog

Detection: the run counts as a hit when the raw serial log contains the
blocking frame `vfs_setxattr+0x` together with an
`INFO: task ... blocked for more than ...` dump.

| run | hit | time to first dump | serial |
|---|---|---|---|
| run-1 | yes | 181 s | run-1/serial.log |
| run-2 | yes | 205 s | run-2/serial.log |
| run-3 | yes | 198 s | run-3/serial.log |
| run-4 | yes | 182 s | run-4/serial.log |
| run-5 | yes | 197 s | run-5/serial.log |

Result: **5/5**, times 181-205 s.

Each `run-N/` contains `serial.log` (full guest console), `run.log` (harness
log with hashes and timings), `execprog-out.log` (syz-execprog stdout) and
the `HIT` marker.

Note: the reproducer must run with `-procs=1`; `-procs=8` did not reproduce
the hang under our tests.
