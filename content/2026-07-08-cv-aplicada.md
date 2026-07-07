---

title: "Computer Vision: notas técnicas de um pipeline completa"  
slug: cv-aplicada
date: 2026-07-08  
tags: IA, computer vision
author: fabricio

---

## **Computer Vision para avaliação de feridas: notas técnicas de um pipeline end-to-end**

Este post documenta o que aprendi ao implementar um sistema de inferência CV para segmentação, classificação e mensuração métrica de feridas crônicas. O foco é arquitetura, algoritmos e decisões de engenharia.

Este post resume o que estudei, aprendi, e apliquei ao longo desse caminho.

---

## **O problema: de pixels a centímetros**

Avaliar feridas crônicas envolve duas tarefas distintas de visão computacional:

1. **Segmentação** — encontrar exatamente onde está a lesão na imagem.
2. **Classificação** — inferir sinais clínicos (infecção, exsudato, etc.).

Mas há um terceiro desafio, que só aparece quando você coloca o sistema no mundo real: **como converter pixels em centímetros?** Sem escala espacial, uma máscara bonita ainda não responde à pergunta que importa: *qual é a área da ferida?*

Esse detalhe mudou completamente minha forma de pensar sobre CV.

## **1. Arquitetura do sistema**

O pipeline é distribuído em três camadas:

| **Camada** | **Stack** | **Responsabilidade** |
| --- | --- | --- |
| Mobile | React Native / Expo | Captura, upload via presigned URL S3 |
| Backend | NestJS + MongoDB | Auth JWT, metadados clínicos, orquestração |
| CV Service | FastAPI + PyTorch/OpenCV | Inferência, pós-processamento, geometria, calibração |

O serviço CV expõe `POST /infer` (JSON `{ image_url }` ou multipart `file`), autenticado via header `x-api-key`, com cache Redis (`SHA-256` do payload) e rate limiting (60 req/min).

Além dos algoritmos, aprendi a estruturar um serviço de inferência de produção:

- **FastAPI** como microserviço de CV, separado do backend NestJS.
- **Orquestrador** que coordena segmentação, classificação, calibração e medição.
- **Modo debug** que salva artefatos intermediários (01–09 + meta.json) para inspecionar cada etapa.
- **Erros tipados** com mensagens técnicas e amigáveis para o frontend.
- **Cache Redis** e rate limiting para uso em produção.
- **Testes unitários** para detecção ArUco, geometria, pós-processamento e validação.

A lição: em CV aplicada, **observabilidade** importa tanto quanto acurácia. Sem debug visual, você debugga no escuro.

Dois modelos rodam em produção:

- **Segmentação**: U-Net + EfficientNet-B3 → **TorchScript** (`torch.jit.trace`)
- **Classificação**: EfficientNet-B3 (`timm`) → **ONNX opset 17** com batch dinâmico

---

## **2. Treinamento**

### **2.1 Segmentação**

Para segmentação, usei uma U-Net com encoder EfficientNet-B3, pré-treinado no ImageNet, via segmentation_models_pytorch. A escolha faz sentido: EfficientNet oferece um bom equilíbrio entre capacidade e eficiência, e a U-Net é o padrão de ouro para máscaras semânticas.

```
smp.Unet(

encoder_name="efficientnet-b3",

encoder_weights="imagenet",

in_channels=3,

classes=1,

)
```

| **Hiperparâmetro** | **Valor** |
| --- | --- |
| Loss | `DiceLoss(mode="binary")` |
| Optimizer | AdamW, lr = 1e-3 |
| Batch size | 4 |
| Image size | 512×512 |
| Early stopping | patience = 5 (maximize val Dice) |
| Mixed precision | AMP (`GradScaler`) |

**Augmentações** (`albumentations`):

```
A.Resize(512, 512)

A.HorizontalFlip(p=0.5)

A.VerticalFlip(p=0.2)

A.RandomRotate90(p=0.5)

A.ShiftScaleRotate(shift_limit=0.05, scale_limit=0.1, rotate_limit=20, p=0.5)

A.ColorJitter(p=0.3)

A.Normalize()
```


### **O que aprendi no treinamento**

- **Dice Loss** funciona melhor que BCE pura para classes desbalanceadas, feridas geralmente ocupam uma fração pequena da imagem.
- **Mixed precision (AMP)** reduz uso de memória e acelera o treino sem perda perceptível de qualidade.
- **Augmentações** (flip, rotação, color jitter, shift/scale) são essenciais, mas insuficientes se o dataset não refletir condições reais de captura.
- Métricas como **IoU e Dice** no validation set são úteis, mas não garantem nada em fotos mobile com flash, sombra ou compressão JPEG agressiva.

Exportei o modelo para **TorchScript**, o que me ensinou que o caminho treino → produção tem requisitos próprios: normalização idêntica, tamanho de entrada fixo e cuidado com operadores suportados na exportação.

### **2.2 Classificação**

EfficientNet-B3 via `timm`, `BCEWithLogitsLoss`, lr = 3e-4, batch = 8, image size = 384. Métrica: F1.

### **2.3 Export**

```
# TorchScript (seg)

scripted = torch.jit.trace(model, dummy_input)  *# dummy: (1, 3, 512, 512)*

# ONNX (cls)

torch.onnx.export(..., opset_version=17, dynamic_axes={"input": {0: "batch"}})
```

**Constraint crítico**: a normalização e o resize da inferência devem ser idênticos ao treino. Qualquer divergência degrada a distribuição de logits da U-Net e desloca o threshold ótimo de binarização.

---

## **3. Pipeline de inferência (orquestrador)**

Fluxo completo em `run_inference_pipeline()`:

original (HxW) → cap MAX_WORKING_DIMENSION=1536 (INTER_AREA) → preprocess_mobile_rgb (CLAHE + brightness norm + unsharp) → letterbox 512×512 (scale = 512/max(h,w), pad simétrico) → tensor (C,H,W) normalizado → U-Net → prob map (512×512) → unletterbox → working resolution → postprocess_wound_mask (threshold sweep + morfologia + CC) → extract_wound_geometry (PCA + minAreaRect + Feret) → overlay PNG base64

A mesma informação de forma mais bonita:

<img src="/media/diagrama.png" alt="Diagrama de fluxo da análise de feridas" width="1000" height="1000">


### **3.1 Letterbox / Unletterbox**

Dado `target_size = 512`:
```
scale = target_size / max(h, w)

content_w = round(w * scale)

content_h = round(h * scale)

pad_left = (512 - content_w) // 2

pad_top  = (512 - content_h) // 2
```
A máscara inferida é recortada da região `[pad_top:pad_top+content_h, pad_left:pad_left+content_w]` e redimensionada para `(working_h, working_w)` com `INTER_NEAREST` (preserva binarização).

**Por que importa**: resize direto para 512×512 distorce aspect ratio e altera a geometria da ferida. Letterbox mantém proporções; o custo é propagar `LetterboxMeta` (scale, pad_left, pad_top) em todo o pipeline.

### **3.2 Pré-processamento mobile**

Aplicado no espaço **LAB** (preserva crominância):

1. **CLAHE** no canal L: `clipLimit=2.0`, `tileGridSize=(8,8)`
2. **Normalização de brilho**: se `mean(gray) < 90` ou `> 200`, escala por `α = clip(128/mean, 0.85, 1.15)`
3. **Unsharp mask**: `out = 1.25·img - 0.25·GaussianBlur(img, σ=1.0)`

Ativado via `MOBILE_IMAGE_PREPROCESS=true`. Não altera geometria — apenas distribuição de intensidade.

---

## **4. Pós-processamento de máscara**

Entrada: mapa probabilístico `P ∈ [0,1]^{H×W}`. Saída: máscara binária + contorno + `geometry_confidence`.

Em vez de um threshold fixo, implementei um **sweep automático** (0.35, 0.40, 0.45, 0.50, 0.55) que escolhe o valor que gera componentes utilizáveis. Depois:

- Morfologia leve para fechar buracos sem fragmentar a região.
- Análise de **componentes conectados** com filtros de plausibilidade (área, aspect ratio, compactness, solidity).
- Fallback para regiões pequenas mas clinicamente plausíveis quando nenhum componente “estrito” passa nos critérios.

### **4.1 Threshold sweep**

`sweep = [0.35, 0.40, 0.45, 0.50, 0.55]  # default produção`

`binary_t = (GaussianBlur(P, k=3) > t).astype(uint8)`

Para cada `t`, computa score baseado em componentes conectados utilizáveis. Seleciona o threshold com melhor score.

**Observação empírica**: threshold 0.65 (comum em papers) produz máscaras vazias em fotos mobile com logits calibrados para distribuição diferente do val set. Default de produção: **0.40**.


### **4.2 Morfologia**

open:  kernel 3×3, 1 iteração  (remove ruído pontual)

close: kernel 5×5, 1 iteração (fecha buracos internos)

Parâmetros tunáveis via env (`MASK_OPEN_KERNEL`, `MASK_CLOSE_KERNEL`).

### **4.3 Seleção de componentes conectados**

`cv2.connectedComponentsWithStats` → filtros em cascata:

| **Filtro** | **Default** | **Função** |
| --- | --- | --- |
| `max_area_fraction` | 0.50 | Rejeita blob > 50% da imagem |
| `max_aspect_ratio` | 10.0 | Rejeita estruturas filamentares |
| `min_compactness` | 0.04 | `4π·A / P²` — rejeita formas irregulares demais |
| `min_solidity` | 0.15 | `A / A_convex_hull` |
| `min_area_fraction` | 0.0002 | Área mínima relativa |
| `min_dimension_px` | 12 | Dimensão mínima do bbox |

Seleção em **3 níveis**:

1. **Strict** — passa todos os filtros
2. **Relaxed** — filtros parcialmente relaxados
3. **Plausible fallback** — se única candidata razoável, aceita região pequena (`MASK_ALLOW_PLAUSIBLE_FALLBACK=true`)

Isso evita o cenário clássico: modelo acerta parcialmente a ferida, mas CC analysis descarta o único componente válido.

### **4.4 Validação pré-geometria**

Antes de extrair contorno, `validate_wound_presence()` verifica se `prob_max` e `area_fraction@0.5` excedem limites mínimos. Falha → `WoundNotDetectedError` com `prob_max` no payload técnico.

Aprendi que pós-processamento é metade do produto em segmentação médica. O modelo diz onde provavelmente está a ferida; o pipeline decide o que mostrar ao clínico.

---

### **5 CLAHE**

O **Contrast Limited Adaptive Histogram Equalization** melhora contraste local em fotos com sombras ou pele escura/clara desigual. É processamento clássico de CV, mas faz diferença real antes da inferência neural.

---

## 6. Análise de coloração tecidual
Esta foi uma das partes mais interessantes do projeto: traduzir cor percebida em percentuais clínicos sem treinar um classificador de cor separado.

### 6.1 Fundamentação clínica
Na prática de curativos, a coloração do leito da ferida orienta conduta. O sistema mapeia pixels para seis categorias inspiradas na escala RYB (Red-Yellow-Black), estendida com roxo e branco:

| Classe  | Interpretação clínica (heurística)                          |
|----------|-------------------------------------------------------------|
| Vermelho | Granulação — tecido de cicatrização                         |
| Amarelo  | Esfacelo — possível necessidade de desbridamento            |
| Preto    | Necrose/eschar — desbridamento                              |
| Roxo     | Hematoma/equimose perilesional                              |
| Branco   | Tecido pálido/isquêmico, biofilme                           |
| Outros   | Pixels que não se encaixam nas regras                       |

A API retorna isso via WoundColorAnalysis (Pydantic), com valores em percentual (0–100) arredondados a 2 casas:

```
class WoundColorAnalysis(BaseModel):
  vermelho: float   # granulação
  amarelo: float    # esfacelo/infecção
  preto: float      # necrose
  roxo: float       # hematoma
  branco: float     # isquêmico/biofilme
  outros: float

```
No mobile, os percentuais alimentam a tela de nova ferida, o `TissueColorPieChart` e a seção de evolução temporal (`WoundEvolutionSection`), permitindo comparar coloração entre capturas do mesmo paciente.

### 6.2 Pipeline de ROI para coloração
Função: `analyze_wound_colors(image_rgb, mask_prob)`.

Passo 1 — Alinhamento e binarização permissiva

```
mask_aligned = align_mask_to_image(image, mask, "analyze_wound_colors")
binary = (mask_aligned > 0.20).astype(uint8)  # threshold baixo: incluir bordas difusas
```

A análise só é chamada após `validate_wound_presence()`, então o threshold de 0.20 (vs 0.40 da geometria) é intencional: captura toda a região provável de ferimento, incluindo halos de baixa confiança na borda.

Passo 2 — Limpeza morfológica
```
MORPH_CLOSE(kernel 3×3, 1 iter)
MORPH_OPEN(kernel 3×3, 1 iter)
→ largest connected component
```

Passo 3 — Máscara de borda

Remove 1% das margens da imagem para evitar contaminação por pele circundante ou artefatos de borda:

```
margin_h = max(1, int(h * 0.01))
margin_w = max(1, int(w * 0.01))
edge_mask[0:margin_h, :] = 0
edge_mask[h-margin_h:, :] = 0
# idem para colunas
binary = binary * edge_mask
```

Passo 4 — Fallback

Se `wound_pixels == 0` após filtros, reutiliza `mask > 0.15` sem máscara de borda, evita retorno zerado em feridas pequenas ou próximas à borda do frame.

### 6.3 Classificação pixel a pixel (HSV + RGB)
Conversão para HSV OpenCV:
```
image_hsv = cv2.cvtColor(image_rgb, cv2.COLOR_RGB2HSV)
# H ∈ [0, 179], S ∈ [0, 255], V ∈ [0, 255]
# Normalização: h/179, s/255, v/255
```

Para cada pixel `(r,g,b)` e `(h,s,v)` dentro da ROI:
```
intensity = (r + g + b) / 3
max_rgb   = max(r, g, b)
min_rgb   = min(r, g, b)
```
Ordem de decisão (first-match wins — evita ambiguidade):

`Preto → Branco → Roxo → Vermelho → Amarelo → Outros`

Preto (necrose)
```
if v < 0.12 or (intensity < 0.15 and max_rgb < 0.2):
    → preto
elif v < 0.18 and intensity < 0.20 and s < 0.25:
    → preto

```

Prioridade máxima: sombras profundas e tecido necrótico têm valor HSV baixo.

Branco (isquêmico / biofilme)
```
elif s < 0.12 and v > 0.65:
    → branco
elif s < 0.18 and v > 0.75 and max_rgb > 0.7:
    → branco
elif s < 0.20 and v > 0.70 and |r-g| < 0.1 and |g-b| < 0.1:
    → branco  # tons acinzentados claros
```
Baixa saturação + alto valor = tecido pálido ou esbranquiçado.

Roxo (hematoma)
Regras HSV e RGB combinadas:
```
elif 0.68 <= h <= 0.92 and s > 0.25 and v > 0.2:
    → roxo
elif r > 0.35 and b > 0.35 and g < 0.35 and s > 0.18:
    → roxo  # R e B dominantes, G suprimido
```

Vermelho (granulação)
```
elif (h < 0.04 or h > 0.96) and s > 0.35 and v > 0.25:
    → vermelho  # matiz no extremo do círculo H (vermelho puro)
elif r > 0.55 and r > g*1.25 and r > b*1.25 and s > 0.25:
    → vermelho  # dominância de R no RGB
elif r > 0.50 and r > g*1.2 and s > 0.30 and 0.2 < v < 0.5:
    → vermelho  # granulação mais escura
```

Amarelo (esfacelo)
```
elif 0.12 <= h <= 0.28 and s > 0.35 and v > 0.3:
    → amarelo
elif r > 0.45 and g > 0.45 and b < 0.35 and s > 0.25:
    → amarelo  # R+G altos, B baixo
elif 0.28 <= h <= 0.40 and s > 0.30 and g > r and g > b:
    → amarelo  # amarelo-esverdeado (infecção)
elif 0.10 <= h <= 0.25 and s > 0.25 and 0.25 < v < 0.55:
    → amarelo  # esfacelo âmbar/escuro
```

Outros
Pixels que não satisfazem nenhuma regra acima.

### 6.4 Agregação
```
total = len(wound_region)  # pixels na ROI filtrada
color_percentages = {
  'vermelho': (red_count   / total) * 100.0,
  'amarelo':  (yellow_count / total) * 100.0,
  'preto':    (black_count  / total) * 100.0,
  'roxo':     (purple_count / total) * 100.0,
  'branco':   (white_count  / total) * 100.0,
  'outros':   (other_count  / total) * 100.0,
}
# Σ ≈ 100%
```

### 6.5 Limitações e aprendizados técnicos
Iluminação domina crominância. Flash direto, sombra e pele circundante alteram `(h,s,v)` de forma não linear. O código detecta alta variação (`std(RGB) > 40`) e loga aviso, mas ainda não aplica correção adaptativa de thresholds — ponto de melhoria.

Segmentação ≠ coloração. Um pixel classificado como vermelho pode ser pele saudável se a máscara vazou. A máscara de borda (1%) e o largest CC mitigam, mas não eliminam o problema.

Rule-based vs ML. Escolhi regras explícitas porque:

- Interpretabilidade clínica (cada threshold é auditável)
- Zero dependência de dataset rotulado por cor
- Latência desprezível (~O(n) por pixel na ROI)

O custo: sensibilidade a white balance da câmera e calibração de cor do dispositivo. Um classificador treinado em LAB normalizado ou um modelo multi-classe por patch seria o próximo passo natural.

Dual color space é necessário. HSV isola matiz (útil para vermelho/amarelo/roxo), mas falha em tons acromáticos (preto/branco) — daí as regras RGB complementares com dominância de canal.

Threshold de máscara diferente da geometria. Geometria usa binarização conservadora (0.40 + CC strict); coloração usa 0.20 permissivo. Aprendi que otimizar uma única máscara para ambos os fins é subótimo.

## **7. Calibração espacial**

Sem `pixels_per_cm`, área em px² não tem unidade clínica. Implementei duas fontes:

### **7.1 ArUco (default)**

Dictionary: DICT_4X4_50

Marker ID:  0 (estrito)

Size:       2.0 cm (lado físico)

Pipeline de detecção:

1. Grayscale + CLAHE
2. `cv2.aruco.ArucoDetector` (compatível com API legada `detectMarkers`)
3. Filtra `marker_id == 0`; se múltiplos, seleciona o de maior perímetro
4. Ordena cantos: TL → TR → BR → BL
5. `pixels_per_cm = mean(side_lengths_px) / marker_size_cm`

**Correção de perspectiva** (opcional, `ARUCO_APPLY_PERSPECTIVE_CORRECTION=true`):

`src = corners[4×2]  (quadrilátero detectado)`

`side = max(edge)  (lado em px)`

`dst = [[0,0], [side,0], [side,side], [0,side]]`

`H = cv2.getPerspectiveTransform(src, dst)`

`rectified = cv2.warpPerspective(image, H, (side, side))`

`mask' = cv2.warpPerspective(mask, H, ..., INTER_NEAREST)`

Homografia planar assume superfície do marcador (e da ferida) aproximadamente coplanar. 
Violação → erro sistemático em área.

Erros tipados:

- `marker_not_detected`
- `invalid_marker` (ID errado)
- `calibration_failed` (marcador muito pequeno, < ~24 px de lado)

**Retry inteligente**: se ArUco falha na imagem inteira, re-detecta usando preview binário da máscara como ROI hint.

### **7.2 Régua clínica (alternativa)**

Como alternativa, implementei detecção de régua clínica com CV clássica:

1. CLAHE → Canny → contornos
2. Filtra contornos com `aspect_ratio > 4.0` (corpo alongado)
3. `minAreaRect` → eixo principal
4. Extrai strip 1D ao longo do eixo (60% da largura)
5. Perfil de gradiente → detecção de picos periódicos (marcações cm)
6. `pixels_per_cm = median(Δx_entre_picos) / RULER_CM_PER_MAJOR_TICK`

Foi fascinante ver que não precisa de deep learning para tudo. Análise de sinais 1D sobre contornos resolve um problema real com interpretabilidade e baixo custo computacional.

Confiança: CV do espaçamento entre picos + número de ticks + aspect ratio.

### **7.3 Fallback**

Se `ALLOW_ASSUMED_FOV_FALLBACK=true` e calibração falha, usa `default_assumed_calibration(image_side_px)` — escala heurística baseada no FOV. **Não é escala clínica**; flag `calibration.is_clinical_scale()` distingue.


Aprendi que ArUco é elegante na teoria e exigente na prática: o marcador precisa estar visível, plano, bem iluminado e com tamanho mínimo em pixels. Por isso o sistema retorna erros específicos (marker_not_detected, invalid_marker, calibration_failed) em vez de falhar silenciosamente.

---

## **8. Extração geométrica**

Com a máscara binária e a calibração, extraio métricas clínicas:

- **Área** e **perímetro** a partir do contorno real.
- **Comprimento × largura** via PCA sobre os pontos do contorno, mais robusto que `boundingRect` para formas irregulares.
- Referência cruzada com `minAreaRect` e **Feret diameter** no convex hull.
- Dimensão final como **mediana robusta** entre PCA e retângulo rotacionado.

Entrada: contorno `C = {(x_i, y_i)}`, calibração `{pixel_to_cm, confidence}`.

### **8.1 Métricas primárias (contorno real)**

```
area_px       = cv2.contourArea(C)

perimeter_px  = cv2.arcLength(C, closed=True)
```

### **8.2 Dimensões clínicas — PCA 2D**

Centro: `μ = mean(C)`

Matriz de covariância: `Σ = cov(C - μ)`

Decomposição: `Σ = V Λ Vᵀ` via `np.linalg.eigh`

Autovetor `v₁` (maior autovalor) = eixo principal:

```
proj_major = (C - μ) · v₁

major_len  = max(proj_major) - min(proj_major)

proj_minor = (C - μ) · v₂

minor_len  = max(proj_minor) - min(proj_minor)

angle_deg  = atan2(v₁_y, v₁_x) · 180/π
```

### **8.3 Referência cruzada**

- `minAreaRect(C)` → `(rect_major, rect_minor, rect_angle)`
- **Feret diameter** no convex hull: `(feret_max, feret_min)`

Dimensão final:

```
width_px  = median(pca_major, rect_major)

height_px = median(pca_minor, rect_minor)
```

**Motivação**: `cv2.boundingRect` superestima feridas alongadas e rotacionadas. PCA captura orientação; mediana entre PCA e minAreaRect reduz sensibilidade a outliers no contorno.

### **8.4 Conversão métrica**

```
area_cm²      = area_px       × pixel_to_cm²

perimeter_cm  = perimeter_px   × pixel_to_cm

width_cm      = width_px      × pixel_to_cm

height_cm     = height_px     × pixel_to_cm

diameter_cm   = max(width_cm, height_cm)
```

### **8.5 Validação geométrica**

`geometry_confidence = min(calibration_confidence, mask_geometry_confidence)`

Checks de plausibilidade (`GEOMETRY_REQUIRE_VALID=true`):

- área mínima em cm²
- aspect ratio dentro de limites
- solidity/compactness acima de thresholds

Falha → `InvalidGeometryError`.

---

## **9. Taxonomia de erros**

O pipeline retorna erros estruturados (HTTP 422) com `message` (frontend) e `message_technical`:

| **Exception** | **Código** | **Trigger** |
| --- | --- | --- |
| `WoundNotDetectedError` | `no_wound_detected` | `prob_max` baixo, área @ 0.5 < threshold |
| `MarkerNotDetectedError` | `marker_not_detected` | ArUco ausente, `ARUCO_REQUIRED=true` |
| `InvalidMarkerError` | `invalid_marker` | ID ≠ 0 |
| `CalibrationFailedError` | `calibration_failed` | Marcador inválido / escala não clínica |
| `InvalidSegmentationError` | `invalid_segmentation` | CC analysis falhou em todos os níveis |
| `GeometryExtractionError` | `geometry_extraction_failed` | Contorno < 5 pontos |
| `InvalidGeometryError` | `invalid_geometry` | Validação geométrica falhou |

---

## 10. Observabilidade e Debug

O sistema possui um modo avançado de observabilidade para facilitar a inspeção de toda a pipeline de segmentação e análise de cores. Quando `DEBUG_SEGMENTATION=true`, são aplicadas configurações mais permissivas para facilitar a identificação de problemas durante o processamento:

- Filtros de Componentes Conectados (CC) relaxados:
  - `min_solidity = 0.08`
  - `max_aspect_ratio = 20`
- `GEOMETRY_MIN_CONFIDENCE = 0.20`

Para cada requisição é criado o diretório `debug_outputs/{request_id}/`, contendo todos os artefatos intermediários da pipeline:

- `01_original.png`
- `02_...`
- `...`
- `09_overlay.png`
- `meta.json`

O arquivo `meta.json` reúne informações detalhadas do processamento, incluindo:

- Dimensões de cada etapa da pipeline;
- Thresholds utilizados;
- Estatísticas da máscara de probabilidade;
- Métricas de segmentação;
- Diagnósticos completos utilizados pela API.

Além disso, a resposta da API passa a incluir o campo `segmentation_debug`, contendo informações detalhadas para auxiliar na investigação de falhas.

Cada requisição também propaga um `ImagePipelineContext`, que registra automaticamente as dimensões produzidas em cada etapa da pipeline (`shape_log`), permitindo identificar rapidamente desalinhamentos entre a imagem original, a inferência da U-Net e a máscara final.

### 10.1 Exemplo de `shape_log`

```text
original_loaded:           shape=(3024, 4032, 3)
working_capped:            shape=(1152, 1536, 3)
preprocessed_mobile:       shape=(1152, 1536, 3)
inference_letterboxed:     shape=(512, 512, 3)
predicted_mask_inference:  shape=(512, 512)
mask_working_prob:         shape=(1152, 1536)
```

Esses registros são fundamentais para diagnosticar problemas como diferenças de escala, padding incorreto, redimensionamentos inadequados e desalinhamentos entre a máscara segmentada e a imagem original.

### 10.2 Logs da análise de cores

A etapa de análise de cores também possui observabilidade dedicada. Durante a execução são registrados:

- Quantidade de pixels analisados;
- Número de pixels classificados em cada cor;
- Percentuais finais de cada classe.

Exemplo:

```text
🎨 [ColorAnalysis] Pixels analisados: 4821
🎨 [ColorAnalysis] Color counts - Red: 2104, Yellow: 891, Black: 312, ...
🎨 [ColorAnalysis] Final percentages:
{
  'vermelho': 43.6,
  'amarelo': 18.5,
  ...
}
```

Esses logs permitem validar tanto a qualidade da segmentação quanto a consistência da classificação cromática, tornando o processo de depuração significativamente mais eficiente.

---

## **11. Testes automatizados**

Cobertura unitária em:

- `test_aruco_detection.py` — detecção, filtro de ID, cálculo de escala
- `test_ruler_detection.py` — picos periódicos, aspect ratio
- `test_measurement_geometry.py` — PCA, Feret, conversão cm
- `test_segmentation_postprocess.py` — threshold sweep, CC fallback
- `test_pipeline_validation.py` — presença de ferida, erros tipados

---

## **12. Lições técnicas consolidadas**

**1. Domain shift mobile >> ganho de arquitetura**

Val Dice/IoU em dataset limpo não prediz comportamento em fotos 12 MP com flash, JPEG agressivo e glare. O pipeline de inferência (CLAHE, threshold sweep, CC fallback) compensa mais do que trocar encoder de B3 para B4.

**2. Threshold é parâmetro de produção, não de treino**

O sweep `[0.35…0.55]` é configurável via env. Diferentes dispositivos/câmeras exigem calibração empírica do threshold sem retreinar.

**3. DL + CV clássica são complementares**

| **Tarefa** | **Abordagem** |
| --- | --- |
| Segmentação semântica | U-Net (DL) |
| Pré-processamento | CLAHE, unsharp (CV clássica) |
| Binarização robusta | Threshold sweep + morfologia + CC |
| Escala métrica | ArUco / régua (CV clássica) |
| Dimensões clínicas | PCA + minAreaRect (álgebra linear) |
| Correção de perspectiva | Homografia (projective geometry) |

**4. Propagação de coordenadas é a fonte #1 de bugs**

Letterbox → inferência → unletterbox → homografia (opcional) → overlay. Cada transformação precisa de metadados reversíveis. `assert_image_mask_aligned()` valida shape/dtype antes de geometria.

**5. Fail-fast com diagnóstico**

Retornar `prob_max=0.12` em `WoundNotDetectedError` é mais útil que retornar máscara vazia silenciosamente. Permite tuning de threshold e detecção de model drift em produção.

**6. Export ≠ treino**

TorchScript trace congela o grafo para `(1,3,512,512)`. Qualquer mudança de input size ou normalização exige re-export. ONNX com `dynamic_axes` no batch é mais flexível para classificação.

---

## **Stack**

- PyTorch 2.x + segmentation_models_pytorch + timm
- OpenCV 4.7+ (aruco, morfologia, homografia)
- FastAPI + uvicorn
- TorchScript (seg) / ONNX opset 17 (cls)
- Redis (cache + rate limit)
- pytest (unit tests)

## **O que eu faria diferente (e o que recomendo estudar)**

1. **Dataset mobile-first** — incluir desde o início fotos com flash, blur, glare, fundos clínicos variados e exemplos negativos (pele sã, tatuagens).
2. **Validar IoU em holdout exclusivamente mobile** — métricas de desktop enganam.
3. **Não subestimar CV clássica** — ArUco, morfologia, CLAHE e análise de contornos são ferramentas maduras que complementam deep learning.
4. **Pensar em falhas desde o design** — o sistema precisa dizer *por que* falhou, não apenas *que* falhou.
5. **Threshold não é hiperparâmetro de treino** — é parâmetro de produto, ajustável por ambiente.