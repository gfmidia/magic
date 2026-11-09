# CLAUDE.md

## Visão geral

O objetivo deste repositório (`magic`) é criar vídeos curtos (short-form) de forma rápida. A base é o **Concat**, um editor de vídeo desktop/mobile em **Rust** que fica em [Concat/](Concat/).

- `Concat/` faz parte deste repositório: o `magic` (origin `github.com/gfmidia/magic`) rastreia todos os arquivos do Concat diretamente, sem repositório aninhado. Commits e pushes são feitos a partir da raiz do `magic`.
- Stack: Rust 1.93.1 (edition 2024), UI em **Slint** (renderizador Skia), composição de vídeo na GPU com **wgpu**, mídia via **FFmpeg 7+** linkado (nada é chamado como processo externo), transcrição com **whisper.cpp**, TTS, e **ONNX Runtime** para recorte de fundo.
- Licença AGPL-3.0-or-later, com CLA ([Concat/CLA.md](Concat/CLA.md)).
- Documentação de referência do próprio projeto: [Concat/src/README.md](Concat/src/README.md) (arquitetura, comandos, API) e [Concat/CONTRIBUTING.md](Concat/CONTRIBUTING.md). Cada crate começa com um bloco `//!` em `src/lib.rs` explicando o que faz.

## Estrutura de pastas

| Pasta | Conteúdo |
|---|---|
| [Concat/src/](Concat/src/) | **Workspace Cargo** (todo o código). `crates/`, `patches/` (Skia e whisper-rs-sys modificados), `scripts/` (builds mobile) |
| [Concat/assets/](Concat/assets/) | Ícones, logos, instaladores (Windows `.iss`/`.wxs`, Linux `nfpm.yaml`, Arch `PKGBUILD`) |
| [Concat/docs/](Concat/docs/) | Atalhos de teclado, MCP, biblioteca de áudio, READMEs traduzidos |
| [Concat/scripts/](Concat/scripts/) | `locales.py` (inventário de strings), `models.py` (manifesto de modelos), scripts de CI e smoke tests |
| [Concat/models/manifest.toml](Concat/models/manifest.toml) | Modelos de IA baixados sob demanda (whisper, recorte, TTS) |
| [Concat/android-native/](Concat/android-native/) | Experimento de app Android nativo (Kotlin + UniFFI) |
| [Concat/.github/workflows/](Concat/.github/workflows/) | CI, build do app, mobile, release, Nix |

## Crates (`Concat/src/crates/`)

| Crate | O que faz |
|---|---|
| `concat-core` | Vocabulário base: tempo exato (`time::Rational`), arena, frames, modelo de timeline, animação. Só std. |
| `concat-media` | Entrada/saída de arquivos via FFmpeg: `probe`, `decode`, `encode`, áudio, waveforms (`peaks`), aceleração por hardware. Único crate que conhece o FFmpeg. |
| `concat-project` | **A edição em si**: modelo do documento (`model.rs`), comandos (`commands/`), undo (`editor.rs`), IO do `concat.json`, SRT (`srt.rs`), importação de CapCut (`capcut.rs`). |
| `concat-render` | Timeline + instante → um frame pronto. `plan.rs` decide o que aparece; `compositor.rs` / `gpu/` desenham com wgpu. Transições em `transitions.rs`. |
| `concat-effects` | Pacotes de efeitos/filtros/transições em `packages/concat.<nome>/` (`effect.toml` + `effect.wgsl`, ou template de filtro FFmpeg para áudio). Descobertos automaticamente pelo `build.rs`. |
| `concat-text` | Títulos viram pixels: shaping, contorno, sombra, placa, em um canvas do tamanho do frame. |
| `concat-vision` | Recorte de fundo (máscaras ONNX, pincéis, Segment Anything), melhoria de imagem, estabilização. |
| `concat-export` | Renderiza a timeline para arquivo: achata clipes, resolve transições, loop frame a frame, mixagem de áudio num filtergraph. |
| `concat-host` | Camada que a janela usa: sessões de projeto, preview do monitor, playback (cpal), jobs, títulos renderizados, cache, logs, modelos, templates. Não toma decisões de edição. |
| `concat-speech` | Transcrição (whisper.cpp) e texto-para-fala (Chatterbox no desktop). |
| `concat-api` | API JSON (requisições → respostas/eventos): projetos, edições, mídia, catálogo, exportação como jobs. |
| `concat-server` | A API num socket: JSON-RPC por TCP/Unix socket, gRPC com feature `grpc`. |
| `concat-cli` | Binário de linha de comando: `probe`, `render`, `api`, `serve`. |
| `concat-mcp` | Servidor MCP para agentes de IA (`concat --mcp`). |
| `concat` | **O app**: a janela do editor em Slint (desktop e iOS). |
| `concat-android` / `concat-mobile` | Entrada Android do app / spike de motor para shell nativo de celular. |

Dependências apontam em um sentido só:

```
concat -> concat-speech -> concat-host -> {export, project, media} -> core
                                       -> render -> core
                                          export -> effects -> project
concat-cli -> concat-server -> concat-api -> concat-host
```

## Compilar e rodar

**Todo comando `cargo` roda em `Concat/src/`**, não na raiz do Concat (é o erro mais comum).

```sh
cd Concat/src
cargo run -p concat                    # janela do editor, debug
cargo dev                              # = cargo run --bin concat --profile quick (otimizado, rebuild rápido; use para iterar)
cargo build --profile app -p concat    # binário de distribuição (LTO fat, panic=abort, sem símbolos)

cargo run -p concat-cli -- probe video.mp4
cargo run -p concat-cli -- render video.mp4 out.mp4 --frames 120
cargo run -p concat-cli -- api '{"method":"catalogue.list","kind":"filter"}'
cargo run -p concat-cli -- serve       # API em 127.0.0.1:7420
```

Checagens antes de PR:

```sh
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
cargo check -p concat                  # obrigatório após mexer em .slint (o macro do Slint gera os bindings)
cargo test -p concat-effects -p concat-export -p concat-render -p concat-project --features gpu   # para shaders/efeitos
python3 scripts/locales.py             # (na raiz do Concat) se mudou texto da interface
```

**Pré-requisitos no Windows** (ver [ci.yml](Concat/.github/workflows/ci.yml)):
- Rust via rustup — a versão 1.93.1 vem de [rust-toolchain.toml](Concat/src/rust-toolchain.toml).
- FFmpeg 7+ build **shared** da BtbN, descompactado; `FFMPEG_DIR` apontando para ele e `<dir>\bin` no `PATH`.
- LLVM com `libclang` (`LIBCLANG_PATH`), cmake e o toolchain C++ do MSVC (para whisper.cpp).
- Opcional: ONNX Runtime compartilhado via `ORT_LIB_LOCATION`.
- Primeiro build leva vários minutos; é normal.

Logs: `<app data>/logs/concat-<quando>.log`; `CONCAT_LOG=debug` aumenta o nível. Use `log::info!/warn!/error!`, nunca `println!`.

## Onde fica cada coisa

Caminhos relativos a `Concat/src/crates/`.

### Interface
- **Telas em Slint**: `concat/ui/` — `app.slint` (raiz), `editor.slint`, `start.slint`, `workspace/` (painéis: mídia, preview, timeline, config), `timeline/` (lanes, trilhas, bandeja de ferramentas), `inspector/` (clip, texto, grading, speed ramp), `dialogs/` (export, legendas, configurações, voz…), `primitives/` (componentes reutilizáveis), `theme/` (claro/escuro), `phone/` (layout de celular).
- **Lógica da janela em Rust**: `concat/src/` — `lib.rs` liga cada callback do Slint ao estado; `studio.rs` + `studio/` guardam o estado e publicam no Slint (`publish.rs`, `menus.rs`, `keyboard.rs`, `inspector.rs`, `stage.rs`…); `panes/` tem um módulo por painel/diálogo no padrão `Msg` → `update`; `platform.rs` concentra as diferenças entre desktop, iOS e Android.
- **Traduções**: `concat/locales/*.json` (ver [TRANSLATING.md](Concat/TRANSLATING.md)).

### Renderização
- **Motor**: `concat-render/src/` — `plan.rs` (o que está na tela, sem pixels), `compositor.rs` e `gpu/` (desenho em wgpu), `transitions.rs`, `upscale.rs`.
- **Efeitos/shaders**: `concat-effects/packages/concat.<nome>/` (`.wgsl` + `effect.toml`); miniaturas em `concat/cards/`.
- **Preview ao vivo**: `concat-host/src/preview.rs` (frame do monitor pausado), `concat-host/src/playback.rs` (reprodução e som), `concat/src/gpu.rs` (device GPU compartilhado entre Slint e o compositor), `concat/src/studio/monitor.rs`, `concat/src/panes/monitor.rs`.
- **Texto/títulos**: `concat-text/src/` + `concat-host/src/titles.rs` (transforma clipes de texto em imagem para o compositor).

### Exportação
- **UI**: `concat/ui/dialogs/export.slint` + `concat/src/panes/export.rs`.
- **Ponte da janela**: `concat-host/src/export.rs` (destino, qualidade, progresso, cancelamento).
- **Motor**: `concat-export/src/` — `flatten.rs` (achata a timeline), `resolve.rs` (transições → sobreposições/fades), `chains.rs`, loop de render em `lib.rs`.
- **Codificação**: `concat-media/src/encode.rs`.
- **Via API**: método `export.run` em `concat-api` (job com eventos `export.progress` / `export.done`).

### Legendas
- **UI**: `concat/ui/dialogs/captions.slint` + `concat/src/panes/captions.rs` (gera de um clipe com áudio ou de um roteiro digitado; posição e tamanho; importa/exporta SRT).
- **Transcrição**: `concat-speech/src/transcribe.rs` (whisper.cpp + Silero VAD, modelos baixados sob demanda).
- **Modelo**: legenda é um clipe de texto — `Command::AddTextClip` e `TextStyle` em `concat-project/src/model.rs` / `commands/`; SRT em `concat-project/src/srt.rs`.
- **Desenho**: `concat-text` + `concat-host/src/titles.rs`.
- **Estilos prontos**: `concat/text-presets/concat.<nome>/preset.toml` (ex.: `bold-caption`, `lower-third`).

## Convenções do código
- Tempo é sempre `concat_core::time::Rational`, nunca `f64`.
- Threads, não `async`. Grafos usam handles (arena), não ponteiros.
- Nenhuma decisão sobre o que uma edição significa fora de `concat-project`; a janela e o host só emitem comandos.
- Arquivos novos começam com o cabeçalho SPDX (`AGPL-3.0-or-later`), como os existentes.
- Código de terceiros só com licença permissiva (MIT/BSD/Apache…); nada GPL/LGPL.
