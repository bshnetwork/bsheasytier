#!/bin/bash
#
# EasyTier Manager - Full Featured
# Cross-distro installer & manager with TOML config & multi-instance
#
# Supports: Debian/Ubuntu, RHEL/CentOS/Fedora, Arch/Manjaro, openSUSE
# UI: whiptail (newt)
#

set -euo pipefail

# =========================================================
# Root check
# =========================================================
if [[ $EUID -ne 0 ]]; then
    echo "This script must be run as root" >&2
    exit 1
fi

# =========================================================
# Constants
# =========================================================
readonly INSTALL_PATH="/etc/easytier"
readonly CONFIG_DIR="${INSTALL_PATH}/configs"
readonly CORE="easytier-core"
readonly CLI="easytier-cli"
readonly EASY_CLIENT="${INSTALL_PATH}/${CLI}"
readonly MAIN_CONFIG="${INSTALL_PATH}/config.toml"
readonly SERVICE_FILE="/etc/systemd/system/easytier.service"
readonly WD_SERVICE="/etc/systemd/system/easytier-watchdog.service"
readonly WD_SCRIPT="${INSTALL_PATH}/monitor.sh"
readonly WD_LOG="${INSTALL_PATH}/monitor.log"
readonly GH_API="https://api.github.com/repos/EasyTier/EasyTier/releases/latest"
readonly WT_TITLE="EasyTier Manager"
readonly SCRIPT_PATH="$(readlink -f "$0")"

# =========================================================
# Colors
# =========================================================
readonly RED=$'\033[0;31m'
readonly GREEN=$'\033[0;32m'
readonly YELLOW=$'\033[0;33m'
readonly CYAN=$'\033[0;36m'
readonly RESET=$'\033[0m'

msg_info() { echo -e "${CYAN}$*${RESET}"; }
msg_ok()   { echo -e "${GREEN}$*${RESET}"; }
msg_warn() { echo -e "${YELLOW}$*${RESET}"; }
msg_err()  { echo -e "${RED}$*${RESET}" >&2; }

# =========================================================
# Whiptail helpers
# =========================================================
wt_msg()   { whiptail --title "$WT_TITLE" --scrolltext --msgbox "$1" 22 78 || true; }
wt_info()  { whiptail --title "$WT_TITLE" --msgbox "$1" 12 78 || true; }
wt_err()   { whiptail --title "$WT_TITLE" --msgbox "$1" 12 78 || true; }

wt_input() {
    local prompt="$1" default="${2:-}" out
    out=$(whiptail --title "$WT_TITLE" --inputbox "$prompt" 10 78 "$default" \
            3>&1 1>&2 2>&3) || return 1
    printf '%s' "$out"
}

wt_password() {
    local prompt="$1" out
    out=$(whiptail --title "$WT_TITLE" --passwordbox "$prompt" 10 78 \
            3>&1 1>&2 2>&3) || return 1
    printf '%s' "$out"
}

wt_yesno() {
    whiptail --title "$WT_TITLE" --yesno "$1" 10 78
}

wt_menu() {
    local prompt="$1"; shift
    local out
    out=$(whiptail --title "$WT_TITLE" --menu "$prompt" 24 78 16 "$@" \
            3>&1 1>&2 2>&3) || return 1
    printf '%s' "$out"
}

wt_checklist() {
    local prompt="$1"; shift
    local out
    out=$(whiptail --title "$WT_TITLE" --checklist "$prompt" 22 78 14 "$@" \
            3>&1 1>&2 2>&3) || return 1
    printf '%s' "$out"
}

# =========================================================
# Distro / platform detection
# =========================================================
DISTRO_ID=""; DISTRO_NAME=""; PKG_MANAGER=""; ARCH=""; PLATFORM=""

detect_distro() {
    if [ -f /etc/os-release ]; then
        # shellcheck disable=SC1091
        . /etc/os-release
        DISTRO_ID="${ID:-unknown}"
        DISTRO_NAME="${PRETTY_NAME:-$ID}"
    fi

    if command -v apt-get >/dev/null 2>&1;   then PKG_MANAGER="apt"
    elif command -v dnf     >/dev/null 2>&1; then PKG_MANAGER="dnf"
    elif command -v yum     >/dev/null 2>&1; then PKG_MANAGER="yum"
    elif command -v pacman  >/dev/null 2>&1; then PKG_MANAGER="pacman"
    elif command -v zypper  >/dev/null 2>&1; then PKG_MANAGER="zypper"
    else PKG_MANAGER="unknown"
    fi
}

detect_platform() {
    if command -v arch >/dev/null 2>&1; then
        PLATFORM=$(arch)
    else
        PLATFORM=$(uname -m)
    fi

    case "$PLATFORM" in
        amd64|x86_64)           ARCH="x86_64" ;;
        arm64|aarch64|*armv8*)  ARCH="aarch64" ;;
        *armv7*)                ARCH="armv7" ;;
        *arm*)                  ARCH="arm" ;;
        mips)                   ARCH="mips" ;;
        mipsel)                 ARCH="mipsel" ;;
        *)                      ARCH="UNKNOWN" ;;
    esac

    if [[ "$ARCH" == "armv7" || "$ARCH" == "arm" ]]; then
        if grep -qi 'half' /proc/cpuinfo 2>/dev/null; then
            ARCH="${ARCH}hf"
        fi
    fi
}

# =========================================================
# Package management
# =========================================================
pkg_name() {
    local cmd="$1"
    case "${PKG_MANAGER}:${cmd}" in
        apt:whiptail)               echo "whiptail" ;;
        dnf:whiptail|yum:whiptail)  echo "newt" ;;
        pacman:whiptail)            echo "libnewt" ;;
        zypper:whiptail)            echo "newt" ;;
        *)                          echo "$cmd" ;;
    esac
}

install_packages() {
    local -a pkgs=("$@")
    [ ${#pkgs[@]} -eq 0 ] && return 0
    case "$PKG_MANAGER" in
        apt)
            DEBIAN_FRONTEND=noninteractive apt-get update -qq
            DEBIAN_FRONTEND=noninteractive apt-get install -y "${pkgs[@]}"
            ;;
        dnf)    dnf install -y "${pkgs[@]}" ;;
        yum)    yum install -y "${pkgs[@]}" ;;
        pacman) pacman -Sy --noconfirm --needed "${pkgs[@]}" ;;
        zypper) zypper --non-interactive install -y "${pkgs[@]}" ;;
        *)      return 1 ;;
    esac
}

install_dependencies() {
    local -a missing_cmds=()
    local c
    for c in whiptail unzip curl openssl bc; do
        command -v "$c" >/dev/null 2>&1 || missing_cmds+=("$c")
    done
    [ ${#missing_cmds[@]} -eq 0 ] && return 0

    local -a missing_pkgs=()
    for c in "${missing_cmds[@]}"; do
        missing_pkgs+=("$(pkg_name "$c")")
    done

    msg_warn "Installing dependencies: ${missing_pkgs[*]}"
    install_packages "${missing_pkgs[@]}" || {
        msg_err "Failed to install: ${missing_cmds[*]}"
        return 1
    }
    msg_ok "Dependencies installed."
}

# =========================================================
# EasyTier install / check  (با GitHub Proxy)
# =========================================================
GH_PROXY=""

check_easytier_installation() {
    [ -f "${INSTALL_PATH}/${CORE}" ] && [ -f "${INSTALL_PATH}/${CLI}" ]
}

get_latest_version() {
    local resp
    resp=$(curl -s --max-time 5 "$GH_API" 2>/dev/null || true)
    echo "$resp" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/' | tr -d '[:space:]' || true
}

install_easytier() {
    local response latest tmp
    response=$(curl -s --max-time 15 "$GH_API" || true)
    latest=$(echo "$response" | grep '"tag_name":' \
              | sed -E 's/.*"([^"]+)".*/\1/' | tr -d '[:space:]' || true)

    if [ -z "$latest" ]; then
        msg_err "Failed to fetch latest EasyTier version."
        return 1
    fi

    msg_info "Installing EasyTier ${latest}..."

    mkdir -p "$INSTALL_PATH" "$CONFIG_DIR"
    tmp=$(mktemp -d /tmp/easytier.XXXXXX)
    trap "rm -rf -- '$tmp'" EXIT

    local base_url="https://github.com/EasyTier/EasyTier/releases/download/${latest}/easytier-linux-${ARCH}-${latest}.zip"
    local url="${GH_PROXY}${base_url}"

    if ! curl --progress-bar -L "$url" -o "$tmp/easytier.zip"; then
        msg_err "Download failed from: ${url}"
        return 1
    fi
    if ! unzip -o "$tmp/easytier.zip" -d "$tmp" >/dev/null; then
        msg_err "Unzip failed."
        return 1
    fi

    local fcore fcli
    fcore=$(find "$tmp" -type f -name "$CORE" | head -n1 || true)
    fcli=$(find  "$tmp" -type f -name "$CLI"  | head -n1 || true)
    if [[ -z "$fcore" || -z "$fcli" ]]; then
        msg_err "Binaries not found in archive."
        return 1
    fi

    install -m 0755 "$fcore" "$INSTALL_PATH/$CORE"
    install -m 0755 "$fcli"  "$INSTALL_PATH/$CLI"

    rm -rf "$tmp"; trap - EXIT
    check_easytier_installation
}

# =========================================================
# TOML config generation
# =========================================================
generate_random_secret() { openssl rand -hex 6; }

# NOTE: برای proxy_network mapping از فرمت دو خطی استفاده می‌کنیم:
#   [[proxy_network]]
#   cidr = "192.168.1.0/24"
#   mapped_cidr = "192.168.2.0/24"
write_main_config() {
    local ip_mode="$1"          # static | dhcp | no-tun
    local ip_address="$2"
    local hostname="$3"
    local instance_name="$4"
    local network_name="$5"
    local network_secret="$6"
    local default_protocol="$7"
    local port="$8"
    local peer_uris="$9"        # space-separated full URIs
    local enable_encryption="${10}"
    local enable_ipv6="${11}"
    local multi_thread="${12}"
    local latency_first="${13}"
    local mtu="${14}"
    local enable_exit_node="${15}"
    local proxy_networks="${16}"   # newline-separated CIDR list
    local vpn_portal="${17}"
    local socks5="${18}"
    local no_listener="${19}"
    local no_tun="${20}"
    local external_node="${21}"
    local log_dir="${22}"
    local config_server="${23}"
    local relay_whitelist="${24}"

    {
        echo "# EasyTier configuration - generated by EasyTier Manager"
        echo "# Generated: $(date -Iseconds)"
        echo ""
        echo "instance_name = \"${instance_name}\""
        echo "hostname = \"${hostname}\""

        case "$ip_mode" in
            static) echo "ipv4 = \"${ip_address}\"" ;;
            dhcp)   echo "dhcp = true" ;;
        esac
        echo ""

        # listeners
        if [[ "$no_listener" == "true" ]]; then
            echo "listeners = []"
        else
            echo "listeners = ["
            echo "  \"${default_protocol}://0.0.0.0:${port}\","
            echo "  \"${default_protocol}://[::]:${port}\","
            if [[ "$vpn_portal" == "true" ]]; then
                echo "  \"wg://0.0.0.0:11011\","
            fi
            echo "]"
        fi
        echo ""

        # rpc portal
        echo "rpc_portal = \"127.0.0.1:15888\""
        echo ""

        # exit nodes (فقط اگر مصرف‌کننده exit node باشد - اینجا ساده نگه می‌داریم)
        echo "exit_nodes = []"
        echo ""

        # network_identity
        echo "[network_identity]"
        echo "network_name = \"${network_name}\""
        echo "network_secret = \"${network_secret}\""
        echo ""

        # peers
        if [[ -n "$peer_uris" ]]; then
            local -a uris=($peer_uris)
            local uri
            for uri in "${uris[@]}"; do
                echo "[[peer]]"
                echo "uri = \"${uri}\""
            done
            echo ""
        fi

        # external node (shared public nodes)
        if [[ "$external_node" == "true" ]]; then
            echo "# Using public shared nodes via -e flag (appended in service)"
            echo ""
        fi

        # proxy_network
        if [[ -n "$proxy_networks" ]]; then
            local line
            while IFS= read -r line; do
                [[ -z "$line" ]] && continue
                # اگر فرمت "src->dst" بود
                if [[ "$line" == *"->"* ]]; then
                    local src="${line%%->*}"
                    local dst="${line##*->}"
                    echo "[[proxy_network]]"
                    echo "cidr = \"${src}\""
                    echo "mapped_cidr = \"${dst}\""
                    echo ""
                else
                    echo "[[proxy_network]]"
                    echo "cidr = \"${line}\""
                    echo ""
                fi
            done <<< "$proxy_networks"
        fi

        # flags
        echo "[flags]"
        echo "default_protocol = \"${default_protocol}\""
        echo "enable_encryption = ${enable_encryption}"
        echo "enable_ipv6 = ${enable_ipv6}"
        echo "latency_first = ${latency_first}"
        echo "enable_exit_node = ${enable_exit_node}"
        echo "no_tun = ${no_tun}"
        echo "use_smoltcp = false"
        if [[ -n "$mtu" ]]; then
            echo "mtu = ${mtu}"
        fi
        if [[ -n "$relay_whitelist" ]]; then
            echo "foreign_network_whitelist = \"${relay_whitelist}\""
        fi
        if [[ -n "$log_dir" ]]; then
            echo "file_log_dir = \"${log_dir}\""
        fi
        if [[ -n "$config_server" ]]; then
            echo "config_server = \"${config_server}\""
        fi
        if [[ -n "$socks5" ]]; then
            echo "socks5 = \"${socks5}\""
        fi
        echo ""
    } > "$MAIN_CONFIG"

    chmod 600 "$MAIN_CONFIG"
}

# =========================================================
# Service file helpers
# =========================================================
write_service_file() {
    local instance_name="$1"   # "default" or custom
    local config_path="$2"
    local service_name="$3"    # easytier.service or easytier@<inst>.service
    local svc_file="/etc/systemd/system/${service_name}"

    cat > "$svc_file" <<EOF
[Unit]
Description=EasyTier Network Service (${instance_name})
After=network.target

[Service]
Type=simple
ExecStart=${INSTALL_PATH}/${CORE} -c ${config_path} $(build_extra_args)
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
    echo "$svc_file"
}

# Extra args that are not easily represented in TOML
EXTRA_ARGS=""
build_extra_args() {
    echo "$EXTRA_ARGS"
}

# =========================================================
# Menu: Connect Tunnel (full wizard with TOML)
# =========================================================
connect_network_pool() {
    # 1. IP mode
    local ip_mode
    ip_mode=$(wt_menu "IP Configuration Mode:" \
        "static" "Manual static IP (recommended for servers)" \
        "dhcp"   "Automatic IP via DHCP (starts at 10.0.0.1)" \
        "no-tun" "No TUN device (SOCKS5/subnet only, no root)") || return 0

    local ip_address="" hostname instance_name
    if [[ "$ip_mode" == "static" ]]; then
        ip_address=$(wt_input "Local IPv4 (e.g. 10.17.17.1):" "") || return 0
        [ -z "$ip_address" ] && { wt_err "IP is required for static mode."; return 0; }
    fi

    hostname=$(wt_input "Hostname:" "$(hostname -s)") || return 0
    [ -z "$hostname" ] && hostname="easytier-node"

    instance_name=$(wt_input "Instance name:" "default") || return 0
    [ -z "$instance_name" ] && instance_name="default"

    # 2. Network identity
    local network_name network_secret default_secret
    network_name=$(wt_input "Network name:" "default") || return 0
    [ -z "$network_name" ] && network_name="default"

    default_secret=$(generate_random_secret)
    network_secret=$(wt_password "Network secret (default: ${default_secret}):")
    [ -z "$network_secret" ] && network_secret="$default_secret"

    # 3. Protocol + Port
    local default_protocol port
    default_protocol=$(wt_menu "Default protocol:" \
        "tcp" "TCP" \
        "udp" "UDP (often more stable)" \
        "ws"  "WebSocket (not recommended in Iran)" \
        "wss" "Secure WebSocket (not recommended in Iran)" \
        "quic" "QUIC (requires --enable-quic-proxy)") || return 0
    case "$default_protocol" in tcp|udp|ws|wss|quic) ;; *) default_protocol="tcp" ;; esac

    port=$(wt_input "Tunnel port:" "2090") || return 0
    [ -z "$port" ] && port="2090"

    # 4. Peers / shared nodes / reverse
    local peer_mode
    peer_mode=$(wt_menu "Peer discovery mode:" \
        "manual" "Manual peer list (comma-separated)" \
        "public" "Use public shared nodes (-e)" \
        "reverse" "Reverse mode (no peers, listen only)") || return 0

    local peer_uris="" external_node="false"
    if [[ "$peer_mode" == "manual" ]]; then
        local raw_peers
        raw_peers=$(wt_input "Peer addresses (IPv4/IPv6, comma-separated):" "") || return 0
        local -a processed=()
        local IFS=',' addr
        for addr in $raw_peers; do
            addr=$(echo "$addr" | xargs)
            [[ -z "$addr" ]] && continue
            if [[ "$addr" == *:* && "$addr" != \[*\] ]]; then
                addr="[$addr]"
            fi
            processed+=("${default_protocol}://${addr}:${port}")
        done
        unset IFS
        [ ${#processed[@]} -gt 0 ] && peer_uris="${processed[*]}"
    elif [[ "$peer_mode" == "public" ]]; then
        external_node="true"
    fi

    # 5. Advanced flags
    local enable_encryption="true" enable_ipv6="true"
    local multi_thread="false" latency_first="false"
    local enable_exit_node="false" no_tun="false"

    wt_yesno "Enable encryption?" || enable_encryption="false"
    wt_yesno "Enable IPv6 support?" || enable_ipv6="false"
    wt_yesno "Enable multi-thread mode?\n(disabled = more stable on some networks)" && multi_thread="true"
    wt_yesno "Enable latency-first mode?\n(recommended for gaming/VoIP)" && latency_first="true"
    wt_yesno "Make this node an Exit Node?\n(route all traffic of peers through this node)" && enable_exit_node="true"

    [[ "$ip_mode" == "no-tun" ]] && no_tun="true"

    # 6. MTU
    local mtu=""
    if wt_yesno "Set custom MTU?\n(default: 1380 non-encrypted / 1360 encrypted)"; then
        mtu=$(wt_input "MTU value:" "1380") || mtu=""
        [[ -z "$mtu" ]] && mtu=""
    fi

    # 7. Subnet proxy
    local proxy_networks=""
    if wt_yesno "Export local subnet(s) to other peers?\n(Subnet Proxy)"; then
        local pn
        pn=$(wt_input "CIDRs (comma-separated).\nFor mapping use: 192.168.1.0/24->192.168.2.0/24" "") || pn=""
        proxy_networks=$(echo "$pn" | tr ',' '\n' | sed '/^\s*$/d')
    fi

    # 8. SOCKS5
    local socks5=""
    if [[ "$ip_mode" == "no-tun" ]] || wt_yesno "Enable built-in SOCKS5 server?"; then
        socks5=$(wt_input "SOCKS5 listen address:" "0.0.0.0:1080") || socks5=""
        [[ -z "$socks5" && "$ip_mode" == "no-tun" ]] && socks5="0.0.0.0:1080"
    fi

    # 9. No listener
    local no_listener="false"
    wt_yesno "Disable all listeners?\n(use only for client-only nodes)" && no_listener="true"

    # 10. VPN Portal (WireGuard)
    local vpn_portal="false"
    wt_yesno "Enable WireGuard VPN Portal?\n(allows WireGuard clients to join)" && vpn_portal="true"

    # 11. Logging
    local log_dir=""
    wt_yesno "Write logs to a file (instead of journald)?" && log_dir="${INSTALL_PATH}/logs"

    # 12. Config server (central management)
    local config_server=""
    if wt_yesno "Connect to a Config Server for central management?"; then
        config_server=$(wt_input "Config Server URL:" "") || config_server=""
    fi

    # 13. Relay whitelist
    local relay_whitelist=""
    if wt_yesno "Restrict relay to specific network names?\n(blank = allow all, '*' = all)"; then
        relay_whitelist=$(wt_input "Network name patterns (space-separated):" "*") || relay_whitelist=""
    fi

    # ---- Generate TOML ----
    write_main_config \
        "$ip_mode" "$ip_address" "$hostname" "$instance_name" \
        "$network_name" "$network_secret" "$default_protocol" "$port" \
        "$peer_uris" "$enable_encryption" "$enable_ipv6" \
        "$multi_thread" "$latency_first" "$mtu" \
        "$enable_exit_node" "$proxy_networks" "$vpn_portal" \
        "$socks5" "$no_listener" "$no_tun" "$external_node" \
        "$log_dir" "$config_server" "$relay_whitelist"

    # ---- Extra command-line args ----
    EXTRA_ARGS=""
    [[ "$multi_thread" == "true" ]] && EXTRA_ARGS+=" --multi-thread"
    [[ "$external_node" == "true" ]] && EXTRA_ARGS+=" -e"
    [[ "$no_listener" == "true" ]] && EXTRA_ARGS+=" --no-listener"
    [[ "$no_tun" == "true" ]] && EXTRA_ARGS+=" --no-tun"
    [[ "$socks5" == "0.0.0.0:1080" && "$no_tun" == "true" ]] && EXTRA_ARGS+=" --socks5 ${socks5}"

    # ---- Service file ----
    local svc_file
    if [[ "$instance_name" == "default" ]]; then
        svc_file=$(write_service_file "default" "$MAIN_CONFIG" "easytier.service")
    else
        # Multi-instance: copy config to named path
        local inst_config="${CONFIG_DIR}/${instance_name}.toml"
        cp "$MAIN_CONFIG" "$inst_config"
        svc_file=$(write_service_file "$instance_name" "$inst_config" "easytier@${instance_name}.service")
    fi

    systemctl daemon-reload
    local sname
    sname=$(basename "$svc_file")
    systemctl enable "$sname" >/dev/null 2>&1 || true

    if systemctl restart "$sname"; then
        wt_info "EasyTier service started successfully.\n\nConfig: ${MAIN_CONFIG}\nService: ${sname}"
    else
        wt_err "Failed to start service. Check: journalctl -u ${sname}"
    fi
}

# =========================================================
# Multi-instance management
# =========================================================
manage_instances() {
    local choice
    choice=$(wt_menu "Multi-Instance Management:" \
        "1" "Create new instance" \
        "2" "List instances" \
        "3" "Remove an instance" \
        "4" "Back") || return 0

    case "$choice" in
        1) create_new_instance ;;
        2) list_instances ;;
        3) remove_instance ;;
        *) return 0 ;;
    esac
}

create_new_instance() {
    local name
    name=$(wt_input "Instance name (alphanumeric, no spaces):" "") || return 0
    [ -z "$name" ] && { wt_err "Name required."; return 0; }
    if [[ ! "$name" =~ ^[a-zA-Z0-9_-]+$ ]]; then
        wt_err "Only alphanumeric, dash and underscore allowed."
        return 0
    fi

    local inst_config="${CONFIG_DIR}/${name}.toml"
    if [[ -f "$inst_config" ]]; then
        wt_err "Instance '${name}' already exists."
        return 0
    fi

    # Simple wizard: ask for base params, write TOML
    local ip_address network_name network_secret port
    ip_address=$(wt_input "Local IPv4:" "") || return 0
    network_name=$(wt_input "Network name:" "default") || return 0
    network_secret=$(wt_password "Network secret:") || return 0
    port=$(wt_input "Port:" "2090") || return 0

    mkdir -p "$CONFIG_DIR"
    cat > "$inst_config" <<EOF
instance_name = "${name}"
hostname = "$(hostname -s)-${name}"
ipv4 = "${ip_address}"
listeners = [
  "tcp://0.0.0.0:${port}",
  "udp://0.0.0.0:${port}",
]
rpc_portal = "127.0.0.1:0"
exit_nodes = []

[network_identity]
network_name = "${network_name}"
network_secret = "${network_secret}"

[flags]
default_protocol = "tcp"
enable_encryption = true
enable_ipv6 = true
EOF
    chmod 600 "$inst_config"

    write_service_file "$name" "$inst_config" "easytier@${name}.service" >/dev/null
    systemctl daemon-reload
    systemctl enable --now "easytier@${name}.service"
    wt_info "Instance '${name}' created and started."
}

list_instances() {
    local out=""
    out+="=== systemd units ===\n"
    out+="$(systemctl list-units 'easytier*' --no-pager --no-legend 2>/dev/null || true)\n\n"
    out+="=== Config files in ${CONFIG_DIR} ===\n"
    out+="$(ls -1 "${CONFIG_DIR}" 2>/dev/null || echo "(none)")\n"
    wt_msg "$out"
}

remove_instance() {
    local name
    name=$(wt_input "Instance name to remove:" "") || return 0
    [ -z "$name" ] && return 0
    if [[ "$name" == "default" ]]; then
        wt_err "Use 'Remove Service' for the default instance."
        return 0
    fi

    local svc="easytier@${name}.service"
    if systemctl list-unit-files "$svc" >/dev/null 2>&1; then
        systemctl disable --now "$svc" >/dev/null 2>&1 || true
    fi
    rm -f "/etc/systemd/system/${svc}" "${CONFIG_DIR}/${name}.toml"
    systemctl daemon-reload
    wt_info "Instance '${name}' removed."
}

# =========================================================
# Terminal display helpers
# =========================================================
run_watch() {
    clear
    msg_info "Running: $*"
    msg_warn "Press Ctrl+C to return."
    echo
    set +e
    "$@"
    set -e
    return 0
}

display_peers()  { run_watch watch -n1 "$EASY_CLIENT" peer; }
display_routes() { run_watch watch -n1 "$EASY_CLIENT" route; }
peer_center()    { run_watch watch -n1 "$EASY_CLIENT" peer-center; }

# =========================================================
# Service management
# =========================================================
restart_easytier_service() {
    if [[ ! -f "$SERVICE_FILE" ]]; then
        wt_err "EasyTier service does not exist."
        return 0
    fi
    if systemctl restart easytier.service; then
        wt_info "EasyTier service restarted successfully."
    else
        wt_err "Failed to restart EasyTier service."
    fi
}

remove_easytier_service() {
    if [[ ! -f "$SERVICE_FILE" ]]; then
        wt_err "EasyTier service does not exist."
        return 0
    fi
    wt_yesno "Are you sure you want to remove the EasyTier service?" || return 0
    systemctl stop    easytier.service >/dev/null 2>&1 || true
    systemctl disable easytier.service >/dev/null 2>&1 || true
    rm -f "$SERVICE_FILE"
    systemctl daemon-reload
    wt_info "EasyTier service removed."
}

show_network_secret() {
    if [[ ! -f "$MAIN_CONFIG" ]]; then
        wt_err "Config file not found."
        return 0
    fi
    local secret
    secret=$(grep -oP 'network_secret\s*=\s*"\K[^"]+' "$MAIN_CONFIG" 2>/dev/null || true)
    if [ -n "$secret" ]; then
        wt_info "Network Secret: ${secret}"
    else
        wt_err "Secret not found."
    fi
}

view_service_status() {
    if [[ ! -f "$SERVICE_FILE" ]]; then
        wt_err "EasyTier service does not exist."
        return 0
    fi
    local out
    out=$(systemctl status easytier.service --no-pager 2>&1 || true)
    wt_msg "$out"
}

# =========================================================
# Config viewer / editor
# =========================================================
view_config() {
    if [[ ! -f "$MAIN_CONFIG" ]]; then
        wt_err "Config file not found."
        return 0
    fi
    local content
    content=$(cat "$MAIN_CONFIG")
    wt_msg "$content"
}

# =========================================================
# GitHub proxy settings
# =========================================================
set_gh_proxy() {
    local current="${GH_PROXY:-(direct)}"
    local new
    new=$(wt_input "GitHub proxy URL (e.g. https://ghfast.top/)\nLeave empty for direct connection.\nCurrent: ${current}" "$GH_PROXY") || return 0
    GH_PROXY="$new"
    # persist in a small file
    echo "GH_PROXY=\"${GH_PROXY}\"" > /etc/easytier/.ghproxy
    chmod 600 /etc/easytier/.ghproxy
    wt_info "GitHub proxy set to: ${GH_PROXY:-(direct)}"
}

load_gh_proxy() {
    if [[ -f /etc/easytier/.ghproxy ]]; then
        # shellcheck disable=SC1091
        . /etc/easytier/.ghproxy
        GH_PROXY="${GH_PROXY:-}"
    fi
}

# =========================================================
# Watchdog
# =========================================================
view_watchdog_status() {
    if systemctl is-active --quiet easytier-watchdog.service 2>/dev/null; then
        echo "running"
    else
        echo "not running"
    fi
}

stop_watchdog_internal() {
    systemctl disable --now easytier-watchdog.service >/dev/null 2>&1 || true
    rm -f "$WD_SCRIPT" "$WD_LOG" "$WD_SERVICE" >/dev/null 2>&1 || true
    systemctl daemon-reload >/dev/null 2>&1 || true
}

start_watchdog() {
    local ip threshold interval
    ip=$(wt_input "Local IP to monitor:" "") || return 0
    [ -z "$ip" ] && { wt_err "IP is required."; return 0; }

    threshold=$(wt_input "Latency threshold (ms):" "200") || return 0
    threshold=${threshold:-200}

    interval=$(wt_input "Check interval (seconds):" "8") || return 0
    interval=${interval:-8}

    stop_watchdog_internal

    cat > "$WD_SCRIPT" <<EOF
#!/bin/bash
IP_ADDRESS="${ip}"
LATENCY_THRESHOLD=${threshold}
CHECK_INTERVAL=${interval}
SERVICE_NAME="easytier.service"
LOG_FILE="${WD_LOG}"

restart_service() {
    local t
    t=\$(date +"%Y-%m-%d %H:%M:%S")
    if systemctl restart "\$SERVICE_NAME"; then
        echo "\$t: restarted OK" >> "\$LOG_FILE"
    else
        echo "\$t: restart FAILED" >> "\$LOG_FILE"
    fi
}

avg_latency() {
    local -a lats=()
    local line
    while IFS= read -r line; do
        lats+=("\$line")
    done < <(ping -c 3 -W 2 -i 0.2 "\$IP_ADDRESS" 2>/dev/null \
             | grep 'time=' | sed -n 's/.*time=\([0-9.]*\) ms.*/\1/p')
    local total=0 count=\${#lats[@]}
    local l
    for l in "\${lats[@]}"; do
        total=\$(echo "\$total + \$l" | bc)
    done
    if [ \$count -gt 0 ]; then
        echo "scale=2; \$total / \$count" | bc
    else
        echo 0
    fi
}

while true; do
    avg=\$(avg_latency)
    if [ "\$avg" = "0" ]; then
        echo "\$(date +"%F %T"): ping failed, restarting" >> "\$LOG_FILE"
        restart_service
    else
        int=\${avg%.*}
        if [ "\$int" -gt "\$LATENCY_THRESHOLD" ]; then
            echo "\$(date +"%F %T"): avg \$avg ms > \$LATENCY_THRESHOLD ms, restarting" >> "\$LOG_FILE"
            restart_service
        fi
    fi
    sleep "\$CHECK_INTERVAL"
done
EOF
    chmod +x "$WD_SCRIPT"

    cat > "$WD_SERVICE" <<EOF
[Unit]
Description=EasyTier Watchdog Service
After=network.target

[Service]
ExecStart=/bin/bash ${WD_SCRIPT}
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

    systemctl daemon-reload >/dev/null 2>&1 || true
    systemctl enable --now easytier-watchdog.service
    wt_info "Watchdog started."
}

stop_watchdog() {
    if [[ ! -f "$WD_SERVICE" ]]; then
        wt_err "Watchdog service does not exist."
        return 0
    fi
    stop_watchdog_internal
    wt_info "Watchdog stopped and removed."
}

view_logs() {
    if [ -f "$WD_LOG" ]; then
        local out
        out=$(tail -n 200 "$WD_LOG" 2>/dev/null || true)
        wt_msg "$out"
    else
        wt_info "No watchdog logs found."
    fi
}

set_watchdog() {
    local status choice
    status=$(view_watchdog_status)
    choice=$(wt_menu "Watchdog status: ${status}" \
        "1" "Create watchdog service" \
        "2" "Stop & remove watchdog service" \
        "3" "View logs" \
        "4" "Back") || return 0
    case "$choice" in
        1) start_watchdog ;;
        2) stop_watchdog  ;;
        3) view_logs      ;;
        *) return 0 ;;
    esac
}

# =========================================================
# Cron
# =========================================================
delete_cron_job_internal() {
    (crontab -l 2>/dev/null | grep -v "#easytier.service_managed_by_script" || true) \
        | crontab - 2>/dev/null || true
    rm -f "${INSTALL_PATH}/reset.sh" 2>/dev/null || true
}

add_cron_job() {
    local choice rt
    choice=$(wt_menu "Select restart interval:" \
        "1"  "Every 30 minutes" \
        "2"  "Every 1 hour" \
        "3"  "Every 3 hours" \
        "4"  "Every 6 hours" \
        "5"  "Every 9 hours" \
        "6"  "Every 17 hours" \
        "7"  "Every 19 hours") || return 0
    case "$choice" in
        1) rt="*/30 * * * *" ;;
        2) rt="0 * * * *" ;;
        3) rt="0 */3 * * *" ;;
        4) rt="0 */6 * * *" ;;
        5) rt="0 */9 * * *" ;;
        6) rt="0 */17 * * *" ;;
        7) rt="0 */19 * * *" ;;
        *) return 0 ;;
    esac
    delete_cron_job_internal
    local reset_path="${INSTALL_PATH}/reset.sh"
    cat > "$reset_path" <<EOF
#!/bin/bash
systemctl daemon-reload
systemctl restart easytier.service
EOF
    chmod +x "$reset_path"
    ( crontab -l 2>/dev/null || true
      echo "$rt $reset_path #easytier.service_managed_by_script" ) | crontab -
    wt_info "Cron job added."
}

delete_cron_job() {
    delete_cron_job_internal
    wt_info "Cron job removed."
}

set_cronjob() {
    local choice
    choice=$(wt_menu "Cron-job settings:" \
        "1" "Add a new cron job" \
        "2" "Delete existing cron job" \
        "3" "Back") || return 0
    case "$choice" in
        1) add_cron_job ;;
        2) delete_cron_job ;;
        *) return 0 ;;
    esac
}

# =========================================================
# Remove core
# =========================================================
remove_easytier_core() {
    if [[ ! -d "$INSTALL_PATH" ]]; then
        wt_err "EasyTier directory not found."
        return 0
    fi
    if wt_yesno "Remove EasyTier core and entire directory ${INSTALL_PATH}?"; then
        rm -rf "$INSTALL_PATH"
        wt_info "EasyTier core removed."
    fi
}

# =========================================================
# Wrapper script  (et)
# =========================================================
install_wrapper() {
    cat > /usr/local/bin/et <<EOF
#!/bin/bash
exec "${SCRIPT_PATH}" "\$@"
EOF
    chmod +x /usr/local/bin/et
}

# =========================================================
# Main menu
# =========================================================
check_core_status() {
    if check_easytier_installation; then echo "Installed"; else echo "Not Found"; fi
}

display_main_menu() {
    local ver status prompt
    ver=$(get_latest_version)
    status=$(check_core_status)
    prompt="Latest: ${ver:-N/A}    Core status: ${status}

Select an option:"

    wt_menu "$prompt" \
        1  "Connect to the Tunnel Network" \
        2  "Display Peers" \
        3  "Display Routes" \
        4  "Peer-Center" \
        5  "Display Secret Key" \
        6  "View Service Status" \
        7  "Set Watchdog (Auto-Restarter)" \
        8  "Cron-job setting" \
        9  "Restart Service" \
        10 "Remove Service" \
        11 "Remove Core" \
        12 "View / Edit Config (TOML)" \
        13 "Multi-Instance Management" \
        14 "GitHub Proxy Settings" \
        0  "Exit"
}

main_loop() {
    while true; do
        local choice
        if ! choice=$(display_main_menu); then
            exit 0
        fi
        case "$choice" in
            1)
                if check_easytier_installation; then connect_network_pool
                else wt_err "EasyTier not installed."; fi
                ;;
            2)
                if check_easytier_installation; then display_peers
                else wt_err "EasyTier not installed."; fi
                ;;
            3)
                if check_easytier_installation; then display_routes
                else wt_err "EasyTier not installed."; fi
                ;;
            4)
                if check_easytier_installation; then peer_center
                else wt_err "EasyTier not installed."; fi
                ;;
            5) show_network_secret ;;
            6) view_service_status ;;
            7) set_watchdog ;;
            8) set_cronjob ;;
            9) restart_easytier_service ;;
            10) remove_easytier_service ;;
            11) remove_easytier_core ;;
            12) view_config ;;
            13) manage_instances ;;
            14) set_gh_proxy ;;
            0) exit 0 ;;
            *) wt_err "Invalid option." ;;
        esac
    done
}

# =========================================================
# Entry point
# =========================================================
main() {
    detect_distro
    detect_platform

    if [ "$ARCH" = "UNKNOWN" ]; then
        echo "Unsupported platform: ${PLATFORM}" >&2
        exit 1
    fi
    if [ "$PKG_MANAGER" = "unknown" ]; then
        echo "Unsupported package manager." >&2
        echo "Install manually: unzip curl openssl bc whiptail" >&2
        exit 1
    fi

    install_dependencies || exit 1

    if ! command -v systemctl >/dev/null 2>&1; then
        wt_err "systemd is required. OpenRC is not supported by this script."
        exit 1
    fi

    load_gh_proxy
    install_wrapper

    if ! check_easytier_installation; then
        if wt_yesno "EasyTier is not installed. Install the latest version now?"; then
            if ! install_easytier; then
                wt_err "EasyTier installation failed."
                exit 1
            fi
            wt_info "EasyTier installed successfully."
        else
            exit 0
        fi
    fi

    set +e
    main_loop
}

main "$@"
