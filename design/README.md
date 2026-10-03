# Pilotei — arquivos de design

Protótipo das 19 telas do app. A fonte de verdade é o canvas "Pilotei — Telas do app" no claude.ai: https://claude.ai/artifact/8Ndkpkogppxa8YmM6LCXzM (privado).

- `telas-html/`: as 19 telas em HTML estático. Abra `telas-html/index.html` no navegador. O botão no canto alterna o tema claro/escuro, e os botões levam de uma tela à outra.
- `fonte/`: arquivos originais do canvas (`.dc.html` + `canvas.json`), copiados em 26/09/2026. Servem de referência de layout, cores e textos; não abrem sozinhos no navegador.

Na versão HTML, Assinatura mostra só o estado "teste grátis" e Configurações mostra o Copiloto desativado. Os demais estados estão no canvas.

**Diferença em relação ao canvas (27/09/2026):** Criar conta e Entrar não têm mais campo de senha (decisão `docs/decisoes/0005-autenticacao.md`: login com Google ou e-mail + OTP). Nova tela **Digitar código** (`telas-html/VerificarCodigo.html`), só na versão HTML: 6 campos, "Reenviar em 60 s" e "Trocar e-mail". Vinda de Criar conta (`?de=criar`), leva ao Dashboard do primeiro acesso; vinda de Entrar, ao Dashboard gerencial.

**Sliders (27/09/2026):** em Custos fixos, Metas e Veículo, todo campo numérico virou slider com o valor digitável acima (arrastar para ajuste rápido, tocar no valor para o número exato). Só na versão HTML. Faixas:
- Custos: seguro 0–10.000 R$/ano; IPVA + licenc. 0–2.500 R$/ano; financiamento 0–4.000 R$/mês; aluguel 0–5.000 R$/mês; outros 0–1.000 R$/mês; manutenção e pneus 0,05–5,00 R$/km.
- Metas: R$/km 0,50–5,00; R$/hora 10–100; resultado diário 50–1.000; horas por dia 1–16 (passo 0,5); KM por mês 500–10.000.
- Veículo: ano 1995–2027; KM atual 0–500.000; consumo 5,0–50,0 km/L.
