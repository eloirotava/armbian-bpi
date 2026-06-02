# Armbian BPI

Repositório com workflows do GitHub Actions para compilar imagens Armbian e LibreELEC para placas e TV boxes ARM selecionadas.

## O que este repositório contém

- **Build Armbian Multi-Board**: workflow manual que usa o framework oficial `armbian/build` para gerar imagens Armbian Trixie minimalistas e pacotes de headers compactados.
- **Build LibreELEC S805**: workflow manual para compilar LibreELEC 9.2 para dispositivos S805/MXQ a partir do repositório `dtechsrv/LibreELEC-AML`.
- **Check Runner Specs**: workflow manual para inspecionar CPU, memória, disco, sistema operacional e rede do runner do GitHub Actions.

## Workflows disponíveis

| Workflow | Arquivo | Finalidade |
| --- | --- | --- |
| Build Armbian Multi-Board (Trixie Minimal) Current e Edge | `.github/workflows/imgs.yaml` | Compila imagens Armbian minimalistas e publica artefatos em uma release. |
| Build LibreELEC 9.2 (S805 - MXQ) | `.github/workflows/LE-s805.yml` | Compila uma imagem LibreELEC para S805/MXQ. |
| Check Runner Specs | `.github/workflows/runner-specs.yml` | Mostra informações do runner usado pelo GitHub Actions. |

## Como executar

Todos os workflows são iniciados manualmente por `workflow_dispatch`:

1. Abra a aba **Actions** no GitHub.
2. Escolha o workflow desejado.
3. Clique em **Run workflow**.
4. Aguarde a finalização do job.
5. Baixe os artefatos gerados ou consulte a release criada automaticamente, quando aplicável.

## Placas configuradas para Armbian

O workflow de Armbian está configurado para compilar, na branch `current`, as seguintes placas/dispositivos:

- `bananapi`
- `aml-s9xx-box`
- `rk322x-box`
- `aml-s805-mxq`

As entradas `edge` estão documentadas no workflow, mas permanecem comentadas.

## Saída esperada

- Imagens Armbian compactadas em `.xz`.
- Pacotes `.deb` de headers compactados em `.xz`.
- Imagem LibreELEC compactada em `.img.gz` para o workflow S805.

## Observações

- Os builds podem consumir bastante tempo, CPU, memória e espaço em disco.
- O workflow do LibreELEC remove ferramentas pré-instaladas do runner para liberar espaço antes da compilação.
- O workflow de Armbian usa permissão `contents: write` para criar releases automaticamente.
