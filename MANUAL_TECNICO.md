# Servidor SFTP em Android TV (Termux + OpenSSH)

Transforme uma Android TV / TV Box em um servidor SFTP permanente, usando o **Termux** com **OpenSSH** e um **HD externo** como armazenamento. Pensado como base para um servidor de mídia/IPTV doméstico.

```
Notebook Windows
      │  SFTP / TCP 8022
      ▼
Android TV / TV Box
      │  Termux + sshd
      ▼
HD externo  /storage/<VOLUME>/CLOUD
```

## Sumário

- [Recursos](#recursos)
- [Requisitos](#requisitos)
- [Instalação rápida](#instalação-rápida)
- [Identificando o HD externo](#identificando-o-hd-externo)
- [Testando a escrita no HD](#testando-a-escrita-no-hd)
- [Conectando pelo Windows](#conectando-pelo-windows)
- [Inicialização automática](#inicialização-automática)
- [Estabilizando o IP](#estabilizando-o-ip)
- [Diagnóstico em três camadas](#diagnóstico-em-três-camadas)
- [Troubleshooting](#troubleshooting)
- [Segurança](#segurança)
- [Roadmap](#roadmap)

## Recursos

- Servidor SFTP rodando no Android TV via Termux
- Armazenamento em HD externo USB
- Acesso pelo Windows (PowerShell `sftp` ou WinSCP)
- Início automático com Termux:Boot
- Guia de diagnóstico para rede, serviço e armazenamento

## Requisitos

- Android TV / TV Box com [Termux](https://termux.dev)
- HD externo USB reconhecido pelo Android
- Notebook na mesma rede local
- [Termux:Boot](https://wiki.termux.com/wiki/Termux:Boot), na mesma distribuição do Termux (mesma fonte de instalação)
- WinSCP (opcional, para interface gráfica)

## Instalação rápida

No Termux:

```sh
pkg update && pkg upgrade
pkg install openssh
passwd        # define a senha
whoami        # mostra o usuário (usado no login SFTP)
termux-setup-storage
sshd
pgrep -a sshd # confirma que o servidor está rodando
```

O OpenSSH do Termux usa a porta **8022** por padrão. Para conferir:

```sh
grep -n '^Port' $PREFIX/etc/ssh/sshd_config
```

> Não adicione uma segunda diretiva `Port 8022` se ela já existir.

Verificar se a porta responde localmente:

```sh
pkg install netcat-openbsd
nc -zv 127.0.0.1 8022
```

Descobrir o IP da TV Box:

```sh
ip -4 addr show wlan0   # se wlan0 não existir: ip -4 addr
```

## Identificando o HD externo

```sh
ls -la /storage
ls -la /storage/<VOLUME>/
ls -la /storage/<VOLUME>/CLOUD
```

> O identificador do volume (ex.: `XXXX-XXXX`) é específico de cada HD. Confirme sempre com `ls -la /storage`.

## Testando a escrita no HD

Teste a escrita **antes** de culpar o SFTP:

```sh
touch ~/termux-write-test.txt
touch /storage/<VOLUME>/CLOUD/RYAN/hd-write-test.txt
```

Se o primeiro funciona e o segundo retorna `Permission denied`, o problema é a permissão do Android/volume, não a rede nem o SFTP. Em exFAT/armazenamento externo, as permissões são impostas pela camada de montagem do Android.

Para investigar:

```sh
ls -ldn /storage/<VOLUME>/CLOUD
df -h /storage/<VOLUME>
```

> Evite `chmod 777` ou `chown` indiscriminados.

## Conectando pelo Windows

Teste a porta no PowerShell:

```powershell
Test-NetConnection IP_DA_TV -Port 8022
```

Esperado: `TcpTestSucceeded : True`. Depois:

```powershell
sftp -P 8022 USUARIO@IP_DA_TV
```

Dentro da sessão:

```
pwd
ls
cd /storage/<VOLUME>/CLOUD/RYAN
ls
```

### WinSCP

| Campo     | Valor                          |
|-----------|--------------------------------|
| Protocolo | SFTP                           |
| Host      | IP da TV Box                   |
| Porta     | 8022                           |
| Usuário   | resultado de `whoami`          |
| Senha     | definida com `passwd`          |

O WinSCP não ultrapassa as permissões do Android: se `touch` falha numa pasta, o upload também falhará.

## Inicialização automática

A abordagem recomendada é o **Termux:Boot** iniciando o `sshd` diretamente (mais simples que o `termux-services`).

1. Instale o Termux:Boot e abra-o uma vez.
2. Crie o script:

```sh
mkdir -p ~/.termux/boot
cat > ~/.termux/boot/start-sftp <<'EOF'
#!/data/data/com.termux/files/usr/bin/sh

termux-wake-lock
sleep 10
sshd
EOF
chmod +x ~/.termux/boot/start-sftp
```

3. Teste antes de reiniciar:

```sh
pkill sshd
~/.termux/boot/start-sftp
pgrep -a sshd
```

4. Reinicie a TV Box **sem abrir o Termux** e teste a conexão a partir do notebook.

### Evitando que o Android mate o servidor

- Termux → Bateria → *Sem restrições / Não otimizar*
- Permitir execução em segundo plano
- Ativar Autostart, se a ROM oferecer
- Não deixar o Termux entrar em apps suspensos

## Estabilizando o IP

O IP pode mudar por causa do DHCP (Wi-Fi ou Ethernet). A solução recomendada é uma **reserva DHCP** no roteador:

1. Abra o painel do roteador
2. Procure *DHCP*, *LAN*, *Address Reservation* ou *Static Lease*
3. Associe o MAC da TV Box a um IP fixo da sua rede
4. Reinicie a TV Box e confirme com `ip -4 addr`

> Trocar de Wi-Fi para Ethernet muda o MAC: pode ser necessária uma nova reserva.

## Diagnóstico em três camadas

```
Rede:          Test-NetConnection IP -Port 8022
Serviço:       pgrep -a sshd
Armazenamento: touch /storage/<VOLUME>/CLOUD/RYAN/teste.txt
```

Se rede e serviço funcionam mas o `touch` dá `Permission denied`, o problema é restrição de escrita no armazenamento, não o SFTP.

## Troubleshooting

| Sintoma | Causa / solução |
|---------|-----------------|
| `sv up sshd` → *service directory does not exist* | O serviço do `termux-services` não foi criado. Use Termux:Boot + `sshd` direto. |
| `sshd` não aparece no `pgrep` | Servidor parado. Execute `sshd`. |
| `nc` local funciona, notebook não conecta | Verifique IP, mesma rede, isolamento de Wi-Fi e firewall. |
| IP mudou | DHCP. Faça reserva no roteador. |
| SFTP conecta, upload dá `Permission denied` | Teste `touch` no mesmo caminho: é permissão de armazenamento. |
| `pwd` mostra `/data/data/com.termux/...` | O `cd` anterior falhou; `pwd` só mostra o diretório atual. |

## Segurança

- Não exponha a porta 8022 diretamente à internet sem necessidade
- Use senha forte e exclusiva; nunca compartilhe credenciais
- Faça backup antes de alterar montagem/permissões
- Mantenha Termux e OpenSSH atualizados
- Quando o servidor não for necessário: `pkill sshd`

## Roadmap

- [x] SFTP funcionando no Termux
- [x] Inicialização automática com Termux:Boot
- [ ] Resolver a escrita no HD externo
- [ ] Isolamento de usuários (`RYAN` e `YCE`, cada um confinado à sua pasta)
- [ ] Integração com IPTV

Estrutura de pastas planejada:

```
HD
└── CLOUD
    ├── RYAN
    └── YCE
```

> Criar duas pastas não cria dois usuários isolados: o OpenSSH atual usa o usuário do Termux. O isolamento exigirá usuários virtuais/chroot ou uma solução SFTP com isolamento por usuário, e só deve ser feito depois de resolver a escrita no HD.

## Checklist

- [ ] OpenSSH instalado e senha configurada
- [ ] `sshd` inicia manualmente
- [ ] Porta 8022 responde
- [ ] Notebook conecta por SFTP
- [ ] HD e pasta `CLOUD` identificados
- [ ] Leitura e escrita do HD testadas separadamente
- [ ] Termux:Boot instalado e script executável
- [ ] Termux sem restrição de bateria
- [ ] Reboot testado sem abrir o Termux
- [ ] IP estabilizado por reserva DHCP

## Licença

Defina a licença do projeto (ex.: MIT).
