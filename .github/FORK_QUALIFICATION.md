# Fork qualification

This branch runs the upstream all-feature check and library-test commands on
standard Linux and Windows runners for four exact published fix commits. Each
job checks out the pinned source SHA, then runs its focused integration tests
when present. Failures do not cancel the other matrix jobs.

The workflow uses public Rust dependencies and the existing official checkout
and toolchain actions. It needs no external service token, paid API, artifact
upload or cache action. GitHub provides the normal read-only checkout token.

Push this CI branch after enabling Actions on the fork. The workflow is separate
from fork main and the fix branches; it adds no commit to any submitted patch.
Changing a source SHA requires reviewing and updating the matrix, then running
the jobs again. Fork results qualify only those source SHAs and do not approve
the upstream maintainer's pending workflow runs.

The precision changes remain Draft while the original-shader skinning
discrepancy is unresolved. CPU CI does not establish GPU correctness.
