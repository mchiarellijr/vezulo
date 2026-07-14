# Vezulo — App (frontend)

Protótipo navegável do Vezulo: landing, login, cadastro e painéis de **candidato** e
**empresa** com o sistema de métricas (match IA explicável, funil de vagas, economia de Vezus,
conquistas). É HTML/CSS/JS puro, arquivo único — **sem etapa de build**.

## Estrutura
- `index.html` — a aplicação inteira (self-contained)
- `docs/sistema-metricas.md` — especificação do sistema de métricas (para a Fase 2 no WordPress)

## Publicar na Hostinger (auto-deploy)
1. hPanel → seu site → **Advanced → Git** → *Continue with GitHub* → autorizar.
2. Selecionar este repositório, branch **main**, pasta de destino **public_html**.
3. Pronto: todo push no `main` publica automaticamente.

## Fluxo de trabalho
Claude edita os arquivos → você dá **Push** → a Hostinger publica sozinha.
