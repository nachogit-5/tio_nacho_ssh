# tio_nacho_ssh
#!/bin/bash

# Script TIO NACHO para Termux
# Configura SSH con puertos adicionales, WebSocket y panel de administración

# Colores
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Variables
INSTALL_DIR="$HOME/tio_nacho"
SSH_PORTS=("22" "80" "443" "8888")
WEBSOCKET_PORT="9999"
USER_DB="$INSTALL_DIR/users.db"

# Funciones
show_banner() {
    clear
    echo -e "${BLUE}"
    echo -e "  _______ _____  _____   ____  _   _  ____ "
    echo -e " |__   __|_ _\ \/ / _ \ / __ \| \ | |/ __ \ "
    echo -e "    | |   | | \  / | | | |  | |  \| | |  | | "
    echo -e "    | |   | | /  \ |_| | |__| | |\  | |__| | "
    echo -e "    |_|  |___/_/\_\___/ \____/|_| \_|\____/ "
    echo -e "${NC}"
    echo -e "         ${YELLOW}TERMUX SSH & WEBSOCKET SERVER${NC}"
    echo -e "             ${YELLOW}by TIO NACHO${NC}"
    echo -e ""
}

check_requirements() {
    if ! command -v sshd &>/dev/null; then
        echo -e "${RED}[!] openssh no está instalado. Instalando...${NC}"
        pkg update -y && pkg install openssh -y
    fi

    if ! command -v wget &>/dev/null; then
        echo -e "${RED}[!] wget no está instalado. Instalando...${NC}"
        pkg install wget -y
    fi

    if ! command -v jq &>/dev/null; then
        echo -e "${RED}[!] jq no está instalado. Instalando...${NC}"
        pkg install jq -y
    fi
}

install_websocket() {
    if ! command -v websocat &>/dev/null; then
        echo -e "${YELLOW}[*] Instalando websocat para WebSocket...${NC}"
        wget https://github.com/vi/websocat/releases/download/v1.10.0/websocat.arm64 -O $INSTALL_DIR/websocat
        chmod +x $INSTALL_DIR/websocat
    fi
}

setup_ssh() {
    echo -e "${YELLOW}[*] Configurando SSH...${NC}"
    
    # Configurar contraseña para el usuario actual
    passwd
    
    # Crear directorio de configuración
    mkdir -p $INSTALL_DIR
    
    # Configurar SSHD para múltiples puertos
    echo -e "${YELLOW}[*] Configurando SSHD para puertos: ${SSH_PORTS[@]}${NC}"
    
    for port in "${SSH_PORTS[@]}"; do
        echo -e "${BLUE}[+] Configurando puerto SSH $port${NC}"
        # Agregar puerto adicional
        if ! grep -q "Port $port" /data/data/com.termux/files/usr/etc/ssh/sshd_config; then
            echo "Port $port" >> /data/data/com.termux/files/usr/etc/ssh/sshd_config
        fi
    done
    
    # Habilitar contraseñas en SSH
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/g' /data/data/com.termux/files/usr/etc/ssh/sshd_config
    
    # Iniciar servicio SSH
    sshd
    
    echo -e "${GREEN}[+] SSH configurado en puertos: ${SSH_PORTS[@]}${NC}"
}

setup_websocket() {
    echo -e "${YELLOW}[*] Configurando WebSocket...${NC}"
    
    # Crear servicio simple para WebSocket
    cat > $INSTALL_DIR/start_websocket.sh <<EOF
#!/bin/bash
while true; do
    $INSTALL_DIR/websocat -s $WEBSOCKET_PORT
    sleep 5
done
EOF
    
    chmod +x $INSTALL_DIR/start_websocket.sh
    
    # Iniciar WebSocket en segundo plano
    nohup $INSTALL_DIR/start_websocket.sh > $INSTALL_DIR/websocket.log 2>&1 &
    
    echo -e "${GREEN}[+] WebSocket configurado en puerto $WEBSOCKET_PORT${NC}"
}

create_user_db() {
    if [ ! -f "$USER_DB" ]; then
        echo -e "${YELLOW}[*] Creando base de datos de usuarios...${NC}"
        echo '{"users":[]}' > "$USER_DB"
    fi
}

add_user() {
    create_user_db
    
    read -p "Ingrese nombre de usuario: " username
    read -s -p "Ingrese contraseña: " password
    echo ""
    
    # Verificar si el usuario ya existe
    if jq -e ".users[] | select(.username == \"$username\")" "$USER_DB" >/dev/null; then
        echo -e "${RED}[!] El usuario ya existe${NC}"
        return
    fi
    
    # Generar ID numérico
    user_id=$(jq '.users | length + 1' "$USER_DB")
    
    # Agregar usuario
    jq --arg user "$username" --arg pass "$password" --argjson id "$user_id" \
       '.users += [{"id": $id, "username": $user, "password": $pass}]' "$USER_DB" > tmp.db && mv tmp.db "$USER_DB"
    
    echo -e "${GREEN}[+] Usuario $username agregado con ID $user_id${NC}"
}

list_users() {
    create_user_db
    
    echo -e "${YELLOW}[*] Lista de usuarios:${NC}"
    jq -r '.users[] | "ID: \(.id) | Usuario: \(.username)"' "$USER_DB"
}

remove_user() {
    create_user_db
    
    list_users
    read -p "Ingrese el ID del usuario a eliminar: " user_id
    
    # Eliminar usuario
    jq --argjson id "$user_id" 'del(.users[] | select(.id == $id))' "$USER_DB" > tmp.db && mv tmp.db "$USER_DB"
    
    # Reorganizar IDs
    temp=$(mktemp)
    jq '.users |= map(.id = index + 1)' "$USER_DB" > "$temp" && mv "$temp" "$USER_DB"
    
    echo -e "${GREEN}[+] Usuario con ID $user_id eliminado${NC}"
}

show_status() {
    echo -e "${YELLOW}[*] Estado del servidor TIO NACHO:${NC}"
    echo ""
    
    # Verificar SSH
    echo -e "${BLUE}Servicio SSH:${NC}"
    for port in "${SSH_PORTS[@]}"; do
        if netstat -tuln | grep -q ":$port "; then
            echo -e "  Puerto $port: ${GREEN}ACTIVO${NC}"
        else
            echo -e "  Puerto $port: ${RED}INACTIVO${NC}"
        fi
    done
    
    # Verificar WebSocket
    echo -e "\n${BLUE}Servicio WebSocket:${NC}"
    if netstat -tuln | grep -q ":$WEBSOCKET_PORT "; then
        echo -e "  Puerto $WEBSOCKET_PORT: ${GREEN}ACTIVO${NC}"
    else
        echo -e "  Puerto $WEBSOCKET_PORT: ${RED}INACTIVO${NC}"
    fi
    
    # Mostrar usuarios
    echo -e "\n${BLUE}Usuarios registrados:${NC}"
    list_users
    
    # Mostrar IP
    ip=$(ifconfig | grep -oP 'inet \K[\d.]+' | grep -v "127.0.0.1")
    echo -e "\n${BLUE}Dirección IP:${NC} $ip"
}

main_menu() {
    while true; do
        show_banner
        echo -e "${YELLOW}MENU PRINCIPAL - TIO NACHO${NC}"
        echo -e ""
        echo -e "1. Configurar servidor SSH"
        echo -e "2. Configurar WebSocket"
        echo -e "3. Administrar usuarios"
        echo -e "4. Ver estado del servidor"
        echo -e "5. Salir"
        echo -e ""
        
        read -p "Seleccione una opción [1-5]: " choice
        
        case $choice in
            1)
                setup_ssh
                ;;
            2)
                setup_websocket
                ;;
            3)
                user_menu
                ;;
            4)
                show_status
                read -p "Presione Enter para continuar..."
                ;;
            5)
                echo -e "${GREEN}[+] Saliendo del script TIO NACHO${NC}"
                exit 0
                ;;
            *)
                echo -e "${RED}[!] Opción inválida${NC}"
                sleep 1
                ;;
        esac
    done
}

user_menu() {
    while true; do
        show_banner
        echo -e "${YELLOW}ADMINISTRACIÓN DE USUARIOS${NC}"
        echo -e ""
        echo -e "1. Agregar usuario"
        echo -e "2. Listar usuarios"
        echo -e "3. Eliminar usuario"
        echo -e "4. Volver al menú principal"
        echo -e ""
        
        read -p "Seleccione una opción [1-4]: " choice
        
        case $choice in
            1)
                add_user
                read -p "Presione Enter para continuar..."
                ;;
            2)
                list_users
                read -p "Presione Enter para continuar..."
                ;;
            3)
                remove_user
                read -p "Presione Enter para continuar..."
                ;;
            4)
                return
                ;;
            *)
                echo -e "${RED}[!] Opción inválida${NC}"
                sleep 1
                ;;
        esac
    done
}

# Inicio del script
show_banner
check_requirements
install_websocket
mkdir -p $INSTALL_DIR
create_user_db
main_menu
