set -a && source .branch-runtime.local && set +a
source scripts/branch_ssh_password_askpass.sh
setup_branch_ssh_password_askpass
mkdir -p "$HOST_SHARED_DATA_DIR"
sshfs "${REMOTE_SSH_TARGET}:/srv/enterprise-rag/data" "$HOST_SHARED_DATA_DIR" -o allow_other
make require-shared-data-visibility
