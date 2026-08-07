# Armbian BPI

Este repositório centraliza automações do GitHub Actions para gerar, publicar e inspecionar builds de imagens ARM usadas em placas Banana Pi e em TV boxes baseadas em Amlogic/Rockchip. A ideia é reduzir o trabalho manual de preparar ambientes de compilação, padronizar os parâmetros usados nos builds e deixar os artefatos disponíveis diretamente pelo GitHub.

O foco atual é:

- compilar imagens **Armbian Trixie minimalistas** com o framework oficial `armbian/build`;
- compilar uma imagem **LibreELEC 9.2** para dispositivos S805/MXQ a partir do projeto `dtechsrv/LibreELEC-AML`;
- disponibilizar um workflow simples para conferir as características do runner antes de investigar falhas de build, limitações de disco ou diferenças de ambiente.

> Este repositório não mantém o código-fonte do Armbian nem do LibreELEC. Os workflows clonam os projetos upstream durante a execução e usam este repositório como ponto de orquestração, documentação e publicação dos resultados.

## Workflows disponíveis

| Workflow | Arquivo | Execução | O que faz | Dispositivos cobertos | Saída principal |
| --- | --- | --- | --- | --- | --- |
| Build Armbian Multi-Board (Trixie Minimal) Current e Edge | `.github/workflows/imgs.yaml` | Manual (`workflow_dispatch`) | Compila imagens Armbian minimalistas e pacotes de headers para cada combinação da matriz. | `bananapi`, `aml-s9xx-box`, `rk322x-box`, `aml-s805-mxq`, nas branches `current` e `edge` (8 combinações). | Artefatos por placa/branch e release na tag informada na execução, com os `.xz` das placas que compilaram. |
| Build LibreELEC 9.2 (S805 - MXQ) | `.github/workflows/LE-s805.yml` | Manual (`workflow_dispatch`) | Compila LibreELEC dentro de um container Ubuntu 18.04 para reproduzir um ambiente compatível com o projeto upstream. | TV boxes S805/MXQ compatíveis com `PROJECT=S805` e `DEVICE=HD18Q`. | Artefato `LibreELEC-S805-Imagem` contendo `*.img.gz`. |

## Objetivo do repositório em mais detalhes

Compilar imagens para placas ARM costuma exigir bastante espaço em disco, dependências específicas, permissões corretas e tempo de CPU. Este repositório organiza esses passos em workflows versionados para que qualquer colaborador com acesso ao GitHub Actions consiga:

1. iniciar um build sem preparar uma máquina local;
2. reutilizar os mesmos parâmetros de compilação em execuções futuras;
3. obter imagens e pacotes de headers como artefatos do GitHub;
4. publicar automaticamente os resultados de Armbian na aba **Releases**;
5. coletar informações do runner quando uma falha parecer relacionada ao ambiente.

## Como executar pela aba Actions

Os dois workflows atuais são manuais. Nenhum deles roda por `push`, `pull_request` ou agendamento.

1. Abra este repositório no GitHub.
2. Entre na aba **Actions**.
3. Escolha um dos workflows no menu lateral.
3.1. No workflow do Armbian, preencha o campo **Versão da Release** (ex.: `v1.0.0`).
4. Clique em **Run workflow**.
5. Selecione a branch que contém a versão do workflow que deseja executar.
6. Clique novamente em **Run workflow** para iniciar.
7. Acompanhe os logs do job até a conclusão.
8. Baixe os artefatos pela página da execução ou, no caso do workflow Armbian, consulte também a aba **Releases**.

### Parâmetros manuais disponíveis

O workflow do Armbian define dois `inputs`:

| Input | Obrigatório | Padrão | Para que serve |
| --- | --- | --- | --- |
| `version` | sim | — | Vira a tag e o nome da release (ex.: `v1.0.0`). |
| `prerelease` | não | `false` | Marca a release como pre-release. |

O do LibreELEC não define `inputs`; o único parâmetro selecionável é a **branch/ref do repositório** que contém o arquivo do workflow.

Os parâmetros de build estão fixos nos arquivos YAML:

- Armbian: matriz de `board` × `branch`, `RELEASE=trixie`, `BUILD_MINIMAL=yes`, `BUILD_DESKTOP=no`, `EXPERT=yes`, `KERNEL_CONFIGURE=no` e `COMPRESS_OUTPUTIMAGE=xz`.
- LibreELEC: `PROJECT=S805`, `DEVICE=HD18Q` e `ARCH=arm`.

Para mudar placa, release, branch do kernel, tipo de imagem ou dispositivo LibreELEC, edite o workflow correspondente em uma branch e execute essa branch pela aba Actions.

## Detalhes de cada workflow

### 1. Build Armbian Multi-Board (`.github/workflows/imgs.yaml`)

Workflow manual para compilar imagens Armbian minimalistas usando `armbian/build`.

#### O que ele faz

1. Libera espaço em disco no runner (builds minimais estouram os ~21 GB livres com facilidade).
2. Habilita QEMU para plataformas `arm` e `arm64` com `docker/setup-qemu-action@v3`.
3. Clona o repositório upstream `armbian/build` no diretório `armbian-build`.
4. Executa `./compile.sh` para cada item da matriz.
5. Ajusta permissões do diretório de saída.
6. Compacta os pacotes `.deb` com `xz -9` (busca recursiva, inclusive subpastas de `output/debs`).
7. Organiza imagem e pacotes em `release-staging/<placa>-<branch>/` e envia como artefato temporário.
8. Em um job final, baixa todos os artefatos `Artefatos-*` e cria uma release do GitHub.

#### Placas e branches cobertas

A matriz é o produto de quatro placas por duas branches, ou seja **8 combinações**:

| Board no workflow | Branches | Observação |
| --- | --- | --- |
| `bananapi` | `current`, `edge` | Placa Banana Pi, config suportada (`.conf`) no Armbian build framework. |
| `aml-s9xx-box` | `current`, `edge` | TV boxes Amlogic S9xx (`.tvb`). |
| `rk322x-box` | `current`, `edge` | TV boxes Rockchip RK322x (`.tvb`). |
| `aml-s805-mxq` | `current`, `edge` | TV boxes Amlogic S805/MXQ (`.tvb`). |

`fail-fast: false` mantém as demais combinações compilando quando uma quebra. As três placas `.tvb` não têm o mesmo nível de suporte do `bananapi`, então é esperado que alguma combinação (`edge`, em especial) falhe — a release sai mesmo assim, só sem ela.

#### Parâmetros fixos de compilação

O workflow passa os seguintes parâmetros para `compile.sh`:

```text
BOARD=<item da matriz>
BRANCH=<item da matriz>
RELEASE=trixie
BUILD_MINIMAL=yes
BUILD_DESKTOP=no
EXPERT=yes
KERNEL_CONFIGURE=no
COMPRESS_OUTPUTIMAGE=xz
```

Isso gera imagens minimalistas, sem desktop, baseadas no Debian Trixie, sem abrir menu interativo de configuração de kernel.

#### Artefatos e releases

Durante o job `armbian-build`, cada combinação publica um artefato com o nome:

```text
Artefatos-<board>-<branch>
```

Cada artefato contém um único diretório `<placa>-<branch>/` com:

- a imagem Armbian (`*.img.xz`), já compactada pelo `COMPRESS_OUTPUTIMAGE=xz`;
- os pacotes `.deb.xz`, renomeados para `<placa>-<branch>__<pacote>.deb.xz`.

O prefixo existe porque assets de release no GitHub são planos: sem ele, pacotes comuns às quatro placas (`armbian-firmware`, `armbian-config`) se sobrescreveriam entre si na release. O `.deb.xz` continua instalável normalmente — basta descompactar e usar `dpkg -i`; o nome do arquivo não importa para o `dpkg`.

Depois que a matriz termina, o job `publish-release` roda com `if: ${{ !cancelled() }}`. **Esse é o ponto central do workflow**: sem isso, uma única placa quebrada marcaria o job da matriz como `failure` e o job de release seria pulado, jogando fora tudo que compilou. Ele baixa os artefatos em `artefatos-finais`, combina os diretórios e cria uma release com:

- tag e nome vindos do input `version` informado na execução;
- `prerelease` conforme o input de mesmo nome;
- corpo listando, por placa, os arquivos publicados e seus tamanhos;
- arquivos anexados vindos de `artefatos-finais/**/*.xz`.

Se nenhuma placa compilar, o job falha explicitamente em vez de criar uma release vazia.

### 2. Build LibreELEC 9.2 S805 (`.github/workflows/LE-s805.yml`)

Workflow manual para compilar LibreELEC 9.2 para TV boxes S805/MXQ.

#### O que ele faz

1. Remove diretórios grandes do runner (`/usr/share/dotnet`, Android SDK, GHC e CodeQL) para liberar espaço.
2. Executa `apt-get clean`.
3. Clona o repositório upstream `dtechsrv/LibreELEC-AML` em `LibreELEC-AML`.
4. Sobe um container `ubuntu:18.04` montando o workspace em `/workspace`.
5. Instala as dependências necessárias dentro do container.
6. Cria o usuário `builder` para compilar sem root.
7. Executa `PROJECT=S805 DEVICE=HD18Q ARCH=arm make image`.
8. Envia o arquivo final como artefato.

#### Dispositivos cobertos

Este workflow está direcionado para dispositivos compatíveis com:

| Parâmetro | Valor | Cobertura esperada |
| --- | --- | --- |
| `PROJECT` | `S805` | Plataforma Amlogic S805. |
| `DEVICE` | `HD18Q` | Perfil usado por boxes MXQ/HD18Q compatíveis. |
| `ARCH` | `arm` | Arquitetura ARM de 32 bits. |

A compatibilidade final depende do hardware específico do TV box, bootloader, memória, armazenamento e device tree esperado pelo projeto LibreELEC-AML.

#### Artefatos

O workflow publica um artefato chamado:

```text
LibreELEC-S805-Imagem
```

Ele inclui os arquivos encontrados em:

```text
LibreELEC-AML/target/*.img.gz
```

O artefato fica disponível na página da execução do workflow, na seção **Artifacts**.


## Pré-requisitos

Para executar os workflows, você precisa de:

- acesso ao repositório no GitHub com permissão para rodar workflows;
- GitHub Actions habilitado no repositório;
- runners hospedados pelo GitHub disponíveis para `ubuntu-latest`;
- cota/minutos de GitHub Actions suficientes para builds longos;
- acesso de rede do runner aos repositórios upstream (`armbian/build` e `dtechsrv/LibreELEC-AML`) e aos servidores de pacotes usados durante a compilação;
- permissões de escrita para o `GITHUB_TOKEN` quando o workflow precisar criar releases.

## Permissões usadas

O workflow Armbian declara:

```yaml
permissions:
  contents: write
```

Essa permissão é necessária para que `softprops/action-gh-release@v2` crie tags/releases e anexe os arquivos gerados. Se a política do repositório ou da organização restringir o token do GitHub Actions para somente leitura, a compilação pode terminar, mas a publicação da release falhará.

O workflow LibreELEC não declara permissões especiais no YAML atual. Ele usa as permissões padrão do `GITHUB_TOKEN` para a execução.

## Limitações conhecidas

- Builds de Armbian e LibreELEC podem demorar bastante em `ubuntu-latest`.
- Espaço em disco é um ponto crítico, especialmente no LibreELEC; por isso o workflow remove ferramentas pré-instaladas antes do build.
- Os workflows dependem de projetos upstream. Mudanças em `armbian/build`, `dtechsrv/LibreELEC-AML`, pacotes APT ou imagens Docker podem quebrar builds sem alteração neste repositório.
- Não há validação automática de boot nos dispositivos reais; um build concluído não garante que a imagem inicialize em todo hardware compatível nominalmente.
- As três placas `.tvb` (`aml-s9xx-box`, `rk322x-box`, `aml-s805-mxq`) não têm suporte de primeira classe no Armbian; falhas em `edge` são esperadas. Por isso a release publica o que compilou em vez de exigir a matriz inteira verde.
- Fora `version` e `prerelease` no workflow Armbian, não há inputs manuais; trocar placa, release ou tipo de imagem exige editar o YAML.
- O build do LibreELEC monta a própria toolchain e pode encostar no teto de 6h do runner hospedado, que mata o job sem gerar imagem.
- O workflow LibreELEC usa `ubuntu:18.04` dentro do Docker por compatibilidade com o projeto, mas essa base é antiga e pode ter limitações de pacotes ou espelhos.
- Artefatos de execução do GitHub Actions expiram conforme a política de retenção configurada no repositório/organização. As releases do workflow Armbian tendem a ser mais persistentes enquanto não forem removidas manualmente.

## Exemplos de uso

### Exemplo 1: gerar uma imagem Armbian para Banana Pi

1. Abra **Actions**.
2. Selecione **Build Armbian Multi-Board (Trixie Minimal) Current e Edge**.
3. Clique em **Run workflow**.
4. Escolha a branch desejada.
5. Inicie a execução.
6. Aguarde todos os itens da matriz terminarem.
7. Baixe o artefato `Artefatos-bananapi-current` na execução ou procure a release `Armbian Trixie Minimal - Build #<número>`.
8. Use a imagem `.xz` conforme o procedimento normal de gravação para a placa.

### Exemplo 2: gerar LibreELEC para um box S805/MXQ

1. Abra **Actions**.
2. Selecione **Build LibreELEC 9.2 (S805 - MXQ)**.
3. Clique em **Run workflow**.
4. Escolha a branch desejada e inicie.
5. Ao final, baixe o artefato `LibreELEC-S805-Imagem`.
6. Extraia ou grave o arquivo `.img.gz` conforme a ferramenta usada para preparar o cartão/armazenamento do dispositivo.

### Exemplo 3: verificar o runner antes de investigar uma falha

1. Abra **Actions** e a execução que falhou.
2. Abra o passo `Liberando espaço no Runner`; ele imprime `df -h /` antes e depois da limpeza.
3. Compare o espaço livre com o de uma execução que deu certo.


### Exemplo 4: testar uma alteração de parâmetros

1. Crie uma branch.
2. Edite o workflow desejado, por exemplo alterando a matriz do Armbian ou os parâmetros `PROJECT`/`DEVICE` do LibreELEC.
3. Faça commit da alteração.
4. Na aba **Actions**, execute o workflow escolhendo essa branch.
5. Confira logs, artefatos e release antes de propor a alteração para a branch principal.

## Troubleshooting

### O build falhou por falta de espaço em disco

- Verifique a saída de `df -h /` no passo `Liberando espaço no Runner`.
- No LibreELEC, confirme se a etapa de limpeza do runner executou corretamente.
- Se necessário, remova mais diretórios não usados ou reduza a matriz do Armbian para compilar menos alvos por execução.

### A release do Armbian não foi criada

- Verifique se o job `armbian-build` terminou com sucesso em todos os itens necessários.
- Confira se o job `publish-release` conseguiu baixar artefatos com o padrão `Artefatos-*`.
- Confirme se o workflow tem `contents: write` e se a configuração do repositório permite escrita pelo `GITHUB_TOKEN`.
- Veja se já existe conflito com a tag `build-<número da execução>`.

### Não encontro os artefatos

- Abra a execução específica em **Actions** e procure a seção **Artifacts**.
- Para Armbian, veja também a aba **Releases**.
- Lembre que artefatos de Actions podem expirar conforme a política de retenção.
- Se a execução falhou antes do passo `upload-artifact`, nenhum artefato será publicado.

### Quero mudar a placa ou o dispositivo

- Armbian: edite a matriz `strategy.matrix.include` em `.github/workflows/imgs.yaml`.
- LibreELEC: ajuste `PROJECT`, `DEVICE` e `ARCH` no comando `make image` em `.github/workflows/LE-s805.yml`.
- Depois rode o workflow manualmente escolhendo a branch alterada.

### A imagem foi gerada, mas não inicializa

- Confirme que o `board`, `PROJECT` ou `DEVICE` corresponde ao hardware real.
- Verifique requisitos específicos de bootloader, device tree, mídia de gravação e fonte de alimentação.
- Consulte os projetos upstream, porque a compatibilidade de hardware é definida principalmente por `armbian/build` e `LibreELEC-AML`.
