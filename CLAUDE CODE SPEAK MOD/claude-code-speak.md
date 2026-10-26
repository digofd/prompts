# Spec: mod "fala" — respostas do Claude Code em voz alta, por sessão

## Objetivo

O Claude Code lê em voz alta cada resposta final (voz `pt-BR-ThalitaMultilingualNeural`, via edge-tts).

Regra principal: **a voz começa DESLIGADA em toda sessão**. Ela só liga na sessão em que o usuário digitar `/fala-on`, e só nessa sessão. Assim, várias sessões abertas não falam ao mesmo tempo.

- `/fala-on`: liga a voz na sessão atual.
- `/fala-off`: desliga a voz na sessão atual.
- O estado fica no `$.state` da sessão. Não use arquivo global nem `$.store`, porque isso vazaria para outras sessões.

Funciona no terminal (CLI) e na extensão do VS Code.

## Pré-requisitos (Windows)

1. Claude Code 2.1.293 ou mais novo, com suporte a mods (hooks em módulo `register.ts`). Testado na CLI 2.1.293 e na extensão do VS Code 2.1.310.
2. Python + edge-tts: `pip install edge-tts`. O comando `edge-tts` precisa estar no PATH.
3. ffmpeg (por causa do `ffplay`): `winget install Gyan.FFmpeg`. O comando `ffplay` precisa estar no PATH.
4. Teste rápido no terminal:
   `edge-tts --voice pt-BR-ThalitaMultilingualNeural --text "teste" | ffplay -nodisp -autoexit -loglevel quiet -i -`

## Arquivos

Crie a pasta `%USERPROFILE%\.claude\mods\fala\` com estes 5 arquivos.

**Atenção ao copiar:** grave cada arquivo exatamente como está aqui. Use um editor ou a ferramenta Write. Não use heredoc do bash: ele pode trocar `\\` por `\`, e isso quebra o mod sem mostrar erro (veja "Se não falar").

### `.claude-plugin/marketplace.json`

~~~json
{
  "name": "mods-locais",
  "owner": { "name": "sande" },
  "plugins": [{ "name": "fala", "source": "./" }]
}
~~~

### `.claude-plugin/plugin.json`

~~~json
{ "name": "fala", "version": "0.2.0", "description": "Lê as respostas em voz alta (edge-tts, voz Thalita) só na sessão onde /fala-on foi chamado", "types": "./types/index.d.ts" }
~~~

### `types/index.d.ts`

Contrato do estado. Sem ele, `claude plugin validate` recusa a chave `fala.ligada`.

~~~ts
export type Ligada = boolean

declare module 'claude-code' {
  interface PluginState {
    fala: { ligada: Ligada }
  }
}
~~~

### `hooks/hooks.json`

~~~json
{ "modules": ["./register.ts"] }
~~~

### `hooks/register.ts`

~~~ts
import { atom, read, update } from 'claude-code'
import type { Register, EngineInterface } from 'claude-code'

import type { Ligada } from '../types'

const VOZ = 'pt-BR-ThalitaMultilingualNeural'

// Fila: cada resposta só toca depois que a anterior terminou.
let fila: Promise<void> = Promise.resolve()
let contador = 0

// Liga/desliga vale só pra sessão atual: toda sessão nova começa muda.
const ligada = atom({ plugin: 'fala', key: 'ligada' } as const, false as Ligada)

function limpar(texto: string): string {
  return texto
    .replace(/```[\s\S]*?```/g, ' ')
    .replace(/`/g, '')
    .replace(/!?\[([^\]]*)\]\([^)]*\)/g, '$1')
    .replace(/[*#]/g, '')
    .replace(/\s+/g, ' ')
    .trim()
}

async function falar($: EngineInterface, texto: string): Promise<void> {
  const temp = await $.env.get('TEMP')
  const nome = `fala-${Date.now()}-${contador++}.txt`
  await $.fs.write(`${temp}\\${nome}`, texto)
  // Áudio vai direto do edge-tts pro ffplay: começa a tocar antes de terminar de gerar.
  await $.process.run(
    ['cmd', '/c', `edge-tts --voice ${VOZ} --file ${nome} | ffplay -nodisp -autoexit -loglevel quiet -i -`],
    { cwd: temp, timeoutMs: 600_000 },
  )
}

export const register: Register = (on) => {
  on('session.start', async ($, e, next) => {
    await $.command.register({ name: 'fala-on', description: 'Liga a leitura em voz alta nesta sessão (voz Thalita)' })
    await $.command.register({ name: 'fala-off', description: 'Desliga a leitura em voz alta nesta sessão' })
    return next(e)
  })

  on('command.run', { command: 'fala-on' }, async ($) => {
    await update($, ligada, () => true)
    return { text: 'Voz ligada nesta sessão.' }
  })

  on('command.run', { command: 'fala-off' }, async ($) => {
    await update($, ligada, () => false)
    return { text: 'Voz desligada nesta sessão.' }
  })

  on('turn.complete', async ($, e, next) => {
    const result = await next(e)
    if (e.agentId || e.reason !== 'answer') return result

    if (!(await read($, ligada))) return result

    const texto = limpar(e.answer)
    if (!texto) return result

    fila = fila.then(() => falar($, texto)).catch(() => {})
    return result
  })
}
~~~

Depois de gravar, confira a linha do `$.fs.write`. Ela precisa ter **duas** barras: `` `${temp}\\${nome}` ``.

## Instalação

1. Valide o mod: `claude plugin validate "%USERPROFILE%\.claude\mods\fala"`. O resultado precisa ser `✔ Validation passed` (os avisos de falta de "author" e de "description" do marketplace podem ser ignorados).
2. Em `%USERPROFILE%\.claude\settings.json`, acrescente estas entradas sem apagar as que já existem. Ajuste o caminho para o usuário da máquina:

~~~json
{
  "enabledPlugins": {
    "fala@mods-locais": true
  },
  "extraKnownMarketplaces": {
    "mods-locais": {
      "source": {
        "source": "directory",
        "path": "C:\\Users\\<USUARIO>\\.claude\\mods\\fala"
      }
    }
  }
}
~~~

3. Cadastre o marketplace e instale o plugin:

   ~~~
   claude plugin marketplace add "%USERPROFILE%\.claude\mods\fala"
   claude plugin install fala@mods-locais
   ~~~

   Sem o `marketplace add`, o install falha com "Marketplace 'mods-locais' not found", mesmo com a entrada no settings.json. Confirme com `claude plugin list`: o `fala@mods-locais` precisa aparecer como `✔ enabled`, com "Read from" apontando para a pasta do mod.
4. Abra uma sessão nova do Claude Code (terminal ou VS Code).

**Para mudar o mod depois:** o plugin é lido direto da pasta `mods\fala`. Basta editar o arquivo e abrir uma sessão nova (ou rodar `/reload-plugins`). O `claude plugin update` não faz nada nesse caso. A cópia em `%USERPROFILE%\.claude\plugins\cache\mods-locais\fala` não é usada.

## O que NÃO pode existir

- `%USERPROFILE%\.claude\commands\fala-on.md` e `fala-off.md`: são da versão antiga, que usava um arquivo global e teria o mesmo nome dos comandos novos. Apague se existirem.
- `%USERPROFILE%\.claude\fala-ligada`: era a flag global da versão antiga. Apague se existir.

## Como verificar (critério de pronto)

1. Abra a sessão A e a sessão B. Mande uma pergunta em cada uma. **Nenhuma pode falar.**
2. Na sessão A, digite `/fala-on`. A resposta deve ser "Voz ligada nesta sessão.".
3. Mande uma pergunta em A: **A fala**. Mande uma pergunta em B: **B continua muda**.
4. Na sessão A, digite `/fala-off` e mande uma pergunta: **A volta a ficar muda**.
5. Feche A e abra uma sessão nova: **ela começa muda**.

## Se não falar

O `.catch(() => {})` esconde os erros, então um problema não aparece na tela. Verifique nesta ordem:

1. **O `/fala-on` respondeu "Voz ligada nesta sessão."?** Se não respondeu, o mod não carregou. Volte à Instalação.
2. **Apareceu um `fala-*.txt` novo em `%TEMP%` depois da resposta?** Se não apareceu, procure um arquivo chamado `Temp${nome}` em `%LOCALAPPDATA%`. Se ele existir, a barra dupla do `register.ts` virou barra simples: corrija a linha e apague esse arquivo.
3. **O `.txt` existe, mas nada tocou?** Rode o teste rápido dos Pré-requisitos. Se ele também não tocar, o problema está no edge-tts, no ffplay ou na internet (o edge-tts precisa de rede).
4. **Ainda nada?** Rode `claude --debug` e procure as linhas `$.fs.write (fala)` e `$.process.run (fala)` no log.
