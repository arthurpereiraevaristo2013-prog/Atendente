# VR Atendente — simulador de loja em VR (Android + VR Box) com hand tracking real

App web (PWA) em **Three.js + MediaPipe Hands (IA)**. Roda no Chrome do Android, em tela dividida para VR Box.

## ⚠️ Leia primeiro (limites honestos)

- **Não é um APK.** O ambiente onde isso foi gerado não tinha acesso a npm/CDN nem ao Android SDK, então não dá para compilar APK aqui. É um app web instalável ("Adicionar à tela inicial"). Se quiser um APK de verdade, veja a seção *Gerar APK* abaixo.
- **Não foi testado em aparelho.** A lógica pura (filtro de suavização, gestos de pinça/punho, projeção das mãos para 3D, troca esquerda/direita) foi testada por script e todos os arquivos passam na checagem de sintaxe. A renderização 3D, a câmera e o MediaPipe só poderão ser validados no seu celular. Espere ajustes finos (alcance das mãos, IPD, distorção).
- **Precisa de internet na 1ª abertura** para baixar Three.js, o runtime do MediaPipe e o modelo `hand_landmarker.task` (CDN jsDelivr + storage.googleapis.com). Depois o service worker guarda em cache; para ficar 100% offline, veja *Offline*.
- **Modelos 3D, texturas e sons são gerados por código** (formas procedurais, texturas em canvas, áudio WebAudio sintetizado). Os personagens são estilizados (low-poly com rosto texturizado), **não** modelos realistas. Sem arquivos de arte externos.
- **Rastreamento:** usa a câmera + IA para achar as 21 juntas de cada mão. A profundidade é *estimada* pelo tamanho aparente da mão (câmera mono), então é boa para alcançar/agarrar, mas não é milimétrica. Rode "Calibrar Hand Tracking".

## Como executar no Android

A câmera só funciona em **HTTPS** (ou `localhost` no próprio aparelho). Opções:

1. **Hospedar (recomendado):** envie esta pasta para GitHub Pages, Netlify, Cloudflare Pages ou similar. Abra o link no Chrome do celular.
2. **Teste rápido do PC ao celular:** na pasta, `python3 -m http.server 8080` e exponha com um túnel HTTPS (`cloudflared tunnel --url http://localhost:8080` ou `ngrok http 8080`). Abra o link HTTPS no celular.
3. No Chrome: menu ⋮ → **Adicionar à tela inicial** (abre em tela cheia, paisagem).

Permissões pedidas: **câmera** (rastrear mãos), sensores de movimento (giroscópio; em alguns Androids está em *Configurações do site → Sensores de movimento*).

## Como jogar

1. Menu → **Calibrar Hand Tracking** (mão aberta a ~35 cm, 3 s) e **Calibrar VR** (IPD/distorção/FOV).
2. **Iniciar jogo**, depois encaixe o celular no VR Box.
3. **Câmera usada para as mãos:** no VR Box o celular fica com a câmera **traseira** voltada para frente, então ela vê suas mãos — é o padrão. Se seu VR Box tem abertura para a frontal, mude em Configurações (ative espelhar).
4. Dicas de uso: boa luz, mãos à frente do celular, fundo sem pele/madeira parecida.

| Gesto | Ação |
|---|---|
| Pinça (polegar+indicador) | Pegar objeto pequeno, notas, puxar a gaveta |
| Punho fechado | Agarrar objeto maior |
| Soltar / abrir a mão | Largar (mantém inércia: dá para arremessar) |
| Indicador esticado | Apontar (raio azul); toque botões com a ponta do dedo |
| Polegar para cima (1,2 s) | Centralizar a câmera |
| Toque curto na tela | Centralizar a câmera |
| Toque longo (1,5 s) | Voltar ao menu |

**Fluxo de atendimento**
1. Cliente entra, vai ao balcão e diz o pedido (balão + monitor).
2. Olhe para as **prateleiras** (esquerda/direita), pegue os itens e coloque na zona **ENTREGAR** do balcão (ou entregue na mão do cliente).
3. Quando completo, o cliente separa o dinheiro. Aperte **COBRAR** no computador (a gaveta abre).
4. Pegue a nota do balcão e solte **dentro da gaveta aberta**.
5. Se houver troco: pegue notas/moedas nos **dispensadores da gaveta** (pince) e coloque na zona **TROCO**. Valor exato = cliente satisfeito + gorjeta.
6. Paciência acaba → cliente vai embora irritado (nota cai). **PRÓXIMO**/sino chama outro cliente.

Modo teste no PC (sem câmera): "Modo teste (mouse)" — mouse = mão, clique = pinça, roda = profundidade, F = punho, Shift = mão esquerda, setas = olhar, C = centralizar.

## Estrutura

```
index.html  style.css  manifest.webmanifest  sw.js  icon.svg
js/main.js         laço principal, fluxos de menu/calibração, resolução dinâmica
js/vr.js           estéreo (tela dividida), distorção de lente, giroscópio, recentralizar
js/handtracking.js câmera → MediaPipe → One Euro → espaço 3D da cabeça; slots esq/dir; recuperação
js/filters.js      filtro One Euro (suaviza parado, não atrasa em movimento)
js/gestures.js     extensão dos dedos, pinça, punho, toque de dedos, apontar, OK, polegar
js/hands3d.js      mãos 3D (21 juntas + ossos + palma) guiadas pelos landmarks reais
js/physics.js      física leve própria (gravidade, atrito, colisões, sono por proximidade)
js/interaction.js  agarrar/soltar/arremessar, botões, gaveta, dispensador de dinheiro
js/world.js        loja: balcão, prateleiras, caixa+gaveta, computador, porta automática, janelas, decoração
js/npc.js          NPCs (8 perfis), animações, expressões, balão de fala, caminhada
js/game.js         pedidos, entrega, cobrança, troco, pontuação, monitor e HUD
js/audio.js        áudio sintetizado com espacialização HRTF
js/textures.js     texturas procedurais; js/ui.js menus/configurações; js/settings.js presets
```

## Hand tracking: o que foi feito

- Duas mãos (`numHands: 2`), 21 juntas cada, delegate **GPU** com fallback para **CPU**.
- **Suavização One Euro** separada para os dedos e para a posição da mão; extrapolação curta (≤60 ms) para baixa latência; mantém a pose por 0,35 s se a detecção falhar (sem "congelar" nem piscar), depois desvanece.
- **Recuperação automática:** ao voltar ao campo de visão, reatribui esquerda/direita por continuidade e reinicia filtros (sem arrastar a mão da posição antiga).
- **Pinça/punho com histerese e confirmação de 2 quadros** (sem disparos por ruído); toque de dedos com o polegar medido por mão.
- **Desempenho:** inferência a ~30 Hz (≈15 Hz no *Modo desempenho*), vídeo 640×480.

## Configurações

Qualidade (baixa/média/alta: sombras, luzes, texturas, NPCs simultâneos), **Modo desempenho**, resolução de renderização (com ajuste automático se o FPS cair), volume, sensibilidade da cabeça e das mãos, suavização, FOV, IPD, distorção das lentes, escolha da câmera, prévia da câmera no jogo. **F3** mostra FPS/latência da IA. Mudar qualidade recarrega o app.

## Offline (opcional)

Baixe e coloque na pasta: `models/hand_landmarker.task` (o app já usa se existir) e troque as URLs do `importmap` em `index.html` por cópias locais de `three.module.js` e `vision_bundle.mjs` (+ pasta `wasm` do `@mediapipe/tasks-vision`, ajustando `WASM_URL` em `js/handtracking.js`). Adicione os arquivos em `CORE` no `sw.js`.

## Gerar APK (opcional)

Hospede o app em HTTPS e use **Bubblewrap** (`npm i -g @bubblewrap/cli && bubblewrap init --manifest=https://SEU-SITE/manifest.webmanifest && bubblewrap build`) ou **PWABuilder.com** para empacotar como APK/AAB (TWA). Alternativa: Capacitor (`npx cap add android`) apontando para esta pasta.

## Solução de problemas

- *Câmera negada / tela preta de mãos:* HTTPS obrigatório; libere a câmera nas permissões do site.
- *Mãos espelhadas ou no lado errado:* troque a câmera/espelhar em Configurações.
- *Mão "longe demais/perto demais":* refaça Calibrar Hand Tracking e ajuste Sensibilidade das mãos.
- *Imagem curva ou borrada nas lentes:* Calibrar VR (IPD e Distorção).
- *FPS baixo:* Modo desempenho + qualidade Baixa + resolução 50–60%.
