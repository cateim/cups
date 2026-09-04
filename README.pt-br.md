# Servidor de Impressão CUPS - Imagem Docker Multi-Arquitetura 🖨️🐳

![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/CaTeIM/cups/build.yml?branch=master&style=for-the-badge)
![Docker Hub Pulls](https://img.shields.io/docker/pulls/cateim/cups?style=for-the-badge)
![Docker Image Size](https://img.shields.io/docker/image-size/cateim/cups/latest?style=for-the-badge)

_[🇺🇸 Read in English](https://github.com/CaTeIM/cups/blob/master/README.md)_

Esta é uma imagem Docker multi-arquitetura do **[CUPS (Common Unix Printing System)](https://github.com/OpenPrinting/cups)**, construída sobre as bases mais recentes do **Ubuntu (Rolling)** e **Debian 13 (Trixie)**. O objetivo é fornecer um servidor de impressão com as versões mais recentes do CUPS, prontas para uso e fáceis de implantar em ambientes containerizados.

## 📚 Código-Fonte

Este projeto é de código aberto. O `Dockerfile`, o script de inicialização e o workflow de build do GitHub Actions estão todos disponíveis no repositório do projeto.

➡️ **[Repositório no GitHub: CaTeIM/cups](https://github.com/CaTeIM/cups)**

## 🐳 Tags Disponíveis

Este repositório constrói duas "trilhas" de imagem, Ubuntu e Debian. As tags vêm
em duas classes, e a diferença importa em produção.

**Tags móveis** sempre apontam para a publicação mais recente da sua trilha:

| Tag                                  | Base da distro     | Aponta para                               |
| :----------------------------------- | :----------------- | :---------------------------------------- |
| `latest`, `ubuntu`                   | Ubuntu (Rolling)   | build Ubuntu mais recente                 |
| `debian`                             | Debian 13 (Trixie) | build Debian mais recente                 |
| `[versão]-ubuntu`, `[versão]-debian` | qualquer uma       | build mais recente daquela versão do CUPS |

**Tags imutáveis** são cunhadas uma vez e nunca reescritas:

| Formato da tag                       | Exemplo                     |
| :----------------------------------- | :-------------------------- |
| `[versão]-[distro]-[AAAAMMDD].[run]` | `2.4.16-ubuntu-20260903.31` |

_💡 **Fixe uma tag imutável em produção.** Tag móvel é cômoda, mas muda debaixo
de você; a imutável é a única forma de saber exatamente o que está rodando e de
voltar para ela. A lista atual está na
[aba Tags](https://hub.docker.com/r/cateim/cups/tags)._

## ✨ Por que usar esta imagem?

- ✅ **Sempre Atualizado**: Reconstruída do zero todo domingo contra os archives
  vivos do Ubuntu Rolling e do Debian 13. Uma imagem nova é publicada sempre que
  qualquer pacote instalado muda de versão, e não apenas quando o CUPS é
  atualizado, de modo que os patches de segurança do SO realmente chegam até
  você. Quando a reconstrução não encontra novidade, nada é publicado e as tags
  móveis ficam onde estão.

- ✅ **Diz o que mudou**: Toda publicação registra quais pacotes se moveram, de
  qual versão para qual, e as CVEs que os mantenedores citam nos changelogs dos
  pacotes. Isso é gravado na própria imagem como annotations OCI
  (`docker buildx imagetools inspect cateim/cups:latest --raw | jq .annotations`)
  e guardado permanentemente em
  [`.github/builds/`](https://github.com/CaTeIM/cups/tree/master/.github/builds).

- ✅ **Multi-Distro**: Escolha entre uma base Ubuntu (`latest`, `2.x.x-ubuntu`) ou Debian (`2.x.x-debian`), dependendo da sua preferência.

- 🔒 **Segura**: O processo de build inclui a aplicação de todas as atualizações de segurança disponíveis (`apt-get upgrade`).

- 🖨️ **Pronta para Uso**: Traz 19 drivers de impressão (Brother, Epson, Canon,
    Samsung, Oki, Dymo, Lexmark, HP e outros), mais o `hplip` e os catálogos de
    PPD `openprinting-ppds` e `foomatic-db`, então a maioria das impressoras é
    plug and play. Cada driver está listado explicitamente no Dockerfile, então
    um que saia da distro quebra o build em vez de sumir calado da imagem.

- 🚀 **Multi-Arquitetura**: Construída para rodar nativamente em `linux/amd64` (PCs, Servidores Intel/AMD) e `linux/arm64` (Raspberry Pi, Orange Pi 5, etc.).

- 🔧 **Configuração Inteligente**: Possui um script de inicialização que configura um usuário administrador e prepara o CUPS para acesso remoto na primeira execução.

## ⚙️ Como Usar (Exemplo com `docker-compose.yml`)

A forma recomendada de usar esta imagem é com o Portainer Stacks ou `docker-compose`. Crie um arquivo `docker-compose.yml` com o seguinte conteúdo.

> **Nota:** Para impressoras USB funcionarem corretamente (detectar falta de papel, reconexão, etc.), o mapeamento dos volumes `/run/udev` e `/run/dbus` é **essencial**.

```yaml
services:
  cups:
    # 'latest' segue o Ubuntu. Em producao, prefira uma tag imutavel:
    # cateim/cups:2.4.16-ubuntu-20260903.31
    image: cateim/cups:latest
    container_name: cups
    # Acesso irrestrito aos devices do host. Funciona em qualquer lugar, mas e
    # muito mais do que imprimir precisa: veja "Rodando sem privileged".
    privileged: true
    restart: unless-stopped
    environment:
      # Senha do usuario 'admin' na interface web. Obrigatoria.
      - ADMIN_PASSWORD=sua_senha_forte
      - TZ=America/Sao_Paulo
    volumes:
      # --- Configuracao e dados ---
      - /srv/cups/config:/etc/cups
      - /srv/cups/logs:/var/log/cups
      - /srv/cups/spool:/var/spool/cups
      # --- Hardware e sistema (CRITICO PARA USB) ---
      # Acesso fisico as portas USB. Mantenha como bind mount: e ele que faz o
      # hotplug funcionar, porque nos criados depois que o container subiu
      # aparecem por ali.
      - /dev/bus/usb:/dev/bus/usb
      # Conversa com os servicos de sistema do host (resolve erros de ColorManager/DBus)
      - /run/dbus:/run/dbus:ro
      # Deixa o CUPS ver eventos de hardware (reposicao de papel, tampa aberta)
      - /run/udev:/run/udev:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "631:631"
    # 'bridge' com a porta publicada e a escolha portavel. 'host' facilita a
    # descoberta na rede (AirPrint/Bonjour), ao custo do isolamento.
    network_mode: bridge
    hostname: cups
    # Da tempo ao cupsd de gravar job.cache e printers.conf antes do SIGKILL.
    stop_grace_period: 30s
    # O CUPS rotaciona o proprio error_log; isto limita apenas o json-file do
    # Docker, que cresceria sem limite num crash loop.
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### ⚠️ Atualizando a partir de uma imagem anterior a 2026-09

Uma mudança incompatível, e ela só afeta você se usava isso: **o pacote `sudo` e
a regra `admin ALL=(ALL) NOPASSWD:ALL` foram removidos.** Com os dois no lugar,
quem descobrisse a senha da interface web em :631 virava root dentro do
container, e com `privileged: true` isso é root no host.

- `docker exec cups <comando>` roda como root e **não é afetado**, que é como
  esta documentação sempre mandou fazer.
- `docker exec -u admin cups sudo <comando>` deixa de funcionar. Tire o
  `-u admin` e o `sudo`.
- O login na interface web **não é afetado**: quem autoriza é a participação no
  grupo `lpadmin`, que o usuário `admin` continua tendo.

Todo o resto daquela versão só acrescenta: 18 drivers de impressora que a imagem
sempre anunciou mas nunca instalou de fato, e metadados OCI de procedência.
Apesar dos drivers a mais, a imagem ficou cerca de 3% menor, porque saíram
headers de compilação que nada usava.

### 🔑 Administração

- Para acessar a interface web, use o endereço: `https://<IP_DO_SEU_SERVIDOR>:631`
- Para acessar a área de **Administration**, use o login `admin` e a senha que você definiu na variável `ADMIN_PASSWORD`.

## 🔒 Rodando sem `privileged`

O exemplo de compose acima usa `privileged: true` porque funciona em qualquer
lugar, mas é muito mais acesso do que imprimir exige. Das sete coisas que o
`privileged` concede, a impressão USB consome exatamente uma: o acesso irrestrito
ao device cgroup. As outras seis são superfície de ataque.

Para tirá-lo, troque `privileged: true` por:

```yaml
    device_cgroup_rules:
      - 'c 189:* rmw'          # 189 é o major do usbfs (include/linux/usb.h)
    security_opt:
      - no-new-privileges:true
    cap_drop: [ALL]
    cap_add: [SETGID, SETUID, KILL, CHOWN, DAC_OVERRIDE, FOWNER, NET_BIND_SERVICE]
```

Mantenha o bind mount de `/dev/bus/usb`. É ele que faz o hotplug funcionar: nós
criados depois que o container subiu aparecem por ali, enquanto uma entrada
`devices:` é um retrato tirado na criação e quebra na primeira vez que a
impressora é desligada e religada.

Por que isso basta: o udev entrega os nós de impressora como `root:lp` modo
`0664`, e os backends do CUPS rodam como `lp`, então o backend já abre o
dispositivo em leitura e escrita pelos bits do grupo. Confira o seu com:

```sh
stat -c '%n dono=%U grupo=%G modo=%a' /dev/bus/usb/BBB/DDD
```

Se a sua impressora não receber `grupo=lp` (alguns multifuncionais se anunciam
como armazenamento em vez da classe impressora), acrescente uma regra udev no
**host** dando `GROUP="lp"` ao seu `idVendor`/`idProduct`, em vez de voltar para
o `privileged`.

**Duas armadilhas que vale conhecer.** O `device_cgroup_rules` é silenciosamente
ignorado enquanto `privileged: true` estiver ligado, então não dá para combinar
os dois "por segurança": a regra vira letra morta e o teste não prova nada. E,
depois de trocar, teste **desligando e religando a impressora** e imprimindo de
novo, não apenas imprimindo uma vez.

## 🐛 Solução de Problemas (USB e Impressoras "Host-Based")

Se você utiliza impressoras USB que dependem de firmware carregado pelo host (como HP LaserJet P1102, P1005, série 1020, etc.), você pode notar que **desligar e ligar a impressora** faz com que o CUPS pare de responder ou deixe trabalhos como "Retidos".

Isso ocorre porque o Container perde a referência do dispositivo USB quando a conexão elétrica cai.

### Solução Definitiva (Regra UDEV no Host)

Para que o Container reconecte automaticamente sempre que a impressora for reiniciada ou o cabo reconectado, crie uma regra `udev` no seu sistema hospedeiro (não no container).

1.  Descubra o ID da sua impressora com o comando `lsusb` (ex: `03f0:002a`).
2.  Crie o arquivo `/etc/udev/rules.d/99-fix-cups-usb.rules` com o conteúdo abaixo (substituindo pelo seu ID):

```bash
# Reinicia o container CUPS automaticamente ao detectar a conexão da impressora
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="SEU_VENDOR_ID", ATTR{idProduct}=="SEU_PRODUCT_ID", RUN+="/usr/bin/docker restart cups"
```

3.  Recarregue as regras: `udevadm control --reload-rules && udevadm trigger`

### Dica Operacional (Falta de Papel)

Se o papel acabar e o trabalho ficar retido, evite desligar a impressora.

1.  Coloque o papel.
2.  **Abra e feche a tampa do toner/cartucho.**
3.  O volume `/run/udev` mapeado no container detectará o evento e o CUPS retomará a impressão automaticamente.

---

_Este projeto não é oficialmente afiliado à OpenPrinting. Todo o crédito pelo CUPS vai para seus respectivos desenvolvedores._
