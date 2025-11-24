# Pipeline para Otimização Completa de Anúncios

Este documento descreve um fluxo automatizado para ler um link de produto, extrair 100% das informações disponíveis e gerar materiais otimizados (título, descrição, atributos, imagens e vídeo em estilo "clip") alinhados às boas práticas dos marketplaces.

## Objetivos
- Ler a URL do produto e mapear todo o conteúdo público (HTML, metadados, schema.org, texto visível e mídia).
- Identificar lacunas, inconsistências ou pontos de melhoria de acordo com o algoritmo/boas práticas do marketplace (SEO interno, políticas e prioridades de ranqueamento).
- Gerar automaticamente versões aprimoradas de texto, estrutura de atributos e assets visuais.

## Entradas
- `product_url`: link do produto no marketplace.
- `marketplace_policy_profile`: conjunto de regras/pesos para cada marketplace (ex.: mínimo de fotos, preferências de título, limites de caracteres, políticas de claims e de compliance).
- `brand_voice_profile`: tom e persona da marca (opcional, para manter consistência).

## Saídas
1. **Relatório de varredura**
   - Dados extraídos: título, preço, descrição, especificações, avaliações, perguntas e respostas, breadcrumbs/categorias, imagens, vídeos.
   - Métricas de completude: quantos campos obrigatórios/preferenciais estão preenchidos.
   - Alertas de risco: termos proibidos, claims sensíveis, falta de provas sociais, baixa qualidade de imagem.

2. **Conteúdo otimizado**
   - **Título**: estruturado com palavras-chave primárias + diferenciais + variações de modelo/tamanho/cor, respeitando limite de caracteres.
   - **Bullet points/descrição**: em blocos curtos, com CTAs sutis e evidências objetivas (materiais, medidas, certificações, garantia).
   - **Atributos estruturados**: mapa `chave:valor` normalizado (cor, tamanho, SKU, compatibilidades, voltagem etc.).
   - **FAQ**: derivado das dúvidas recorrentes e pontos de fricção identificados.

3. **Assets visuais sugeridos**
   - **Imagens**: roteiro de até 8 fotos com instruções de enquadramento, iluminação, ângulos, contexto de uso e chamadas breves.
   - **Vídeo estilo clip** (vertical ou quadrado): storyboard com 6–10 cenas curtas (1–3s) cobrindo: problema, solução, demonstração rápida, prova social, call-to-action. Indicar legendas, overprints e proporção.
   - Para geração com IA, produzir prompts prontos (ex.: Stable Diffusion, Midjourney, DALL·E) incluindo cenário, iluminação, lente, estilo e restrições do marketplace.

## Etapas do Pipeline
1. **Coleta de dados**
   - Fazer request HTTP e usar um parser HTML para coletar: `<title>`, metatags (`og:*`, `twitter:*`), JSON-LD (`application/ld+json`), microdados, textos visíveis e URLs de mídia.
   - Normalizar URLs de imagens/vídeos e baixar metadados básicos (dimensão, peso, formato).

2. **Extração e normalização**
   - Converter HTML em texto limpo preservando hierarquia (títulos, listas, tabelas).
   - Detectar idioma, categoria e atributos chave usando dicionários/regex + modelo de NLP leve.
   - Consolidar todas as fontes em um objeto `product_snapshot` versionado.

3. **Diagnóstico de qualidade**
   - Regras rápidas: mínimo de fotos, presença de variações, densidade de keywords no título, tamanho de descrição, presença de certificações/garantia, avaliações > X estrelas e N reviews.
   - Detectar claims arriscados (saúde, performance, superlativos absolutos) e sugerir reformulação.

4. **Geração de melhorias**
   - Usar modelo de linguagem para sugerir títulos, bullets e FAQ otimizados, respeitando limites do marketplace e voz da marca.
   - Gerar prompts para imagens e storyboard de vídeo conforme o gap identificado (ex.: faltam close-ups, uso real, escala).

5. **Validação final**
   - Rodar regras de compliance (caracteres especiais, menções proibidas, políticas de devolução/garantia padrão do marketplace).
   - Pontuar a versão proposta e devolver `diff` entre o anúncio atual e o recomendado.

## Considerações de Implementação
- Empacotar cada etapa como serviço isolado (ingestão, análise, geração, validação) para facilitar observabilidade e testes.
- Salvar `product_snapshot` + resultado gerado em banco (ex.: PostgreSQL) para auditoria.
- Registrar origem das decisões (evidência -> sugestão) para transparência com o time de catálogo.
- Permitir execução em lote recebendo uma lista de URLs.

## Turbinar a inteligência e o throughput (1000+ anúncios sem humanos)
- **Orquestração assíncrona e em lote**: usar filas (ex.: RabbitMQ/Kafka) para disparar pipelines em paralelo. Cada etapa vira um worker escalável (ingestão, análise, geração de texto, geração de mídia, validação), permitindo throughput de milhares de URLs/hora.
- **Aceleração de ingestão**: crawling/headless em paralelo (playwright/puppeteer) com cache agressivo de assets e deduplicação por hash de HTML/imagens. Pré-validação de links mortos/redirects evita trabalho ocioso.
- **Inferência e reranqueamento rápidos**: modelos compactos (distil/quantized) para classificação de categorias/idioma e reranqueamento de palavras-chave. Somente o que é lacuna relevante segue para modelos maiores de geração.
- **Geração visual full-automated**:
  - Criação de prompts dinâmicos baseada em atributos detectados (cor, material, uso, público) + persona da marca.
  - Bibliotecas de estilos pré-aprovados por marketplace (fundo neutro, proporção, resolução mínima) e guardrails automáticos (checar rosto/marca d'água/NSFW antes de publicar).
  - Inpainting/upscaling automático para remover fundos inadequados, ruídos ou adicionar contexto de uso; variantes multi-ângulo geradas em lote.
- **Vídeos estilo clip sem intervenção humana**:
  - Storyboard gerado a partir de benefícios/objeções detectados; cenas curtas com sobreposições de texto e call-to-action padrão do marketplace.
  - Sintetização de narração e legendas automáticas; montagem automática (ffmpeg) com cortes rápidos e trilha isenta de copyright.
- **Self-healing e feedback loops**: métricas de clique/conversão/qualidade alimentam reranker de prompts e texto. Conteúdos com baixa performance são reprocessados em background com novas variações.
- **Política-first**: regras de compliance aplicadas antes e depois da geração (ex.: claims proibidos, menções a saúde, garantias). Reprovação gera nova variação automaticamente em vez de fila manual.
- **Observabilidade**: rastreamento de cada anúncio com `trace_id`, dashboards de SLA (tempo por etapa) e alarmes de queda de qualidade. Fácil de provar que rodou “100%” das verificações.

## Fluxo sugerido para 1000 anúncios em minutos
1. **Input**: lista de URLs ou feed CSV/JSON enviado para a API.
2. **Fan-out**: enfileira cada URL; workers de ingestão baixam HTML/mídia em paralelo.
3. **Diagnóstico e priorização**: cálculo rápido de lacunas; anúncios com maiores gaps disparam pipelines completos de geração (texto + imagens + vídeo). Casos quase completos passam apenas por validação.
4. **Geração**: prompts e storyboards automáticos; renderização paralela de imagens (multi-GPU) e clips (pipelines otimizadas com caching de assets e templates de edição).
5. **Validação automática**: checagem de conformidade de mídia (resolução, fundo, brand safety) e texto; regenerar se reprovar.
6. **Publicação/entrega**: devolver pacote pronto (texto + assets) via API ou publicar direto, com logs e diffs.

## Exemplo de Uso (pseudo-API)
```http
POST /v1/product-optimize
{
  "product_url": "https://marketplace.com/p/123",
  "marketplace_policy_profile": "ml_default",
  "brand_voice_profile": "tech_friendly"
}
```
Resposta:
```json
{
  "report": { "completeness": 0.78, "alerts": ["falta vídeo", "título curto"] },
  "optimized": { "title": "...", "description": "...", "attributes": {...}, "faq": ["..."] },
  "visuals": { "images": ["roteiro"], "video_clip": ["storyboard"] }
}
```

