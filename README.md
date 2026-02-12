# robot-prep

lmms-eval + PhysBench 평가 프레임워크.

[lmms-eval](https://github.com/EvolvingLMMs-Lab/lmms-eval)을 서브모듈로 포함하며, [PhysBench](https://github.com/physical-superintelligence-lab/PhysBench) (ICLR 2025) 태스크가 추가되어 있습니다.

---

## 설치

```bash
git clone --recurse-submodules https://github.com/rby011/robot-prep.git
cd robot-prep
pip install -r requirements.txt
```

이미 클론한 경우:

```bash
git submodule update --init --recursive
pip install -r requirements.txt
```

---

## PhysBench 데이터셋 준비

PhysBench는 이미지(3.7 GB)와 비디오(3.7 GB) 파일을 별도로 다운로드해야 합니다.

```bash
export PHYSBENCH_DATA_DIR=/path/to/physbench

# 1. 데이터셋 다운로드
huggingface-cli download USC-GVL/PhysBench \
    --local-dir $PHYSBENCH_DATA_DIR \
    --repo-type dataset

# 2. 미디어 압축 해제
cd $PHYSBENCH_DATA_DIR
unzip image.zip -d image
unzip video.zip -d video
```

디렉터리 구조:

```
$PHYSBENCH_DATA_DIR/
├── test.json
├── image/          ← image.zip 압축 해제
│   └── *.png
└── video/          ← video.zip 압축 해제
    └── *.mp4
```

---

## 평가 실행

### openai_compatible 모델 (OpenAI API 호환 서버)

```bash
export PHYSBENCH_DATA_DIR=/path/to/physbench

python -m lmms_eval \
    --model openai_compatible_chat \
    --model_args \
        model_version=gpt-4o,\
        base_url=https://api.openai.com/v1,\
        api_key=$OPENAI_API_KEY \
    --tasks physbench \
    --batch_size 1 \
    --log_samples \
    --output_path ./results
```

### vLLM 모델

```bash
export PHYSBENCH_DATA_DIR=/path/to/physbench

python -m lmms_eval \
    --model vllm_chat \
    --model_args \
        pretrained=Qwen/Qwen2.5-VL-7B-Instruct,\
        max_num_frames=32 \
    --tasks physbench \
    --batch_size 1 \
    --log_samples \
    --output_path ./results
```

### Qwen3-VL 모델 (로컬 HuggingFace)

```bash
export PHYSBENCH_DATA_DIR=/path/to/physbench

python -m lmms_eval \
    --model qwen3_vl_chat \
    --model_args \
        pretrained=Qwen/Qwen2.5-VL-7B-Instruct,\
        max_num_frames=32 \
    --tasks physbench \
    --batch_size 1 \
    --log_samples \
    --output_path ./results
```

---

## PhysBench 태스크 개요

| 항목 | 내용 |
|---|---|
| 데이터셋 | [USC-GVL/PhysBench](https://huggingface.co/datasets/USC-GVL/PhysBench) |
| 문항 수 | 10,002 (test) |
| 형식 | 4지선다 (A/B/C/D) |
| 미디어 | 비디오 + 이미지 인터리브드 |
| 평가 지표 | 정확도 (overall + task_type / ability_type / sub_type 세부 분류) |

**4대 카테고리**: 물리적 물체 속성 · 물체 관계 · 장면 이해 · 물리 역학
**8 능력 차원**: identify, comparison, static, dynamic, perception, prediction, judgment, reasoning
**19 하위 유형**: number, mass, color, attribute, size, location, depth, distance, movement, temperature, camera, gas, light, collision, throwing, manipulation, fluid, chemistry, others

---

## 구조

```
robot-prep/
├── lmms-eval/                          ← git submodule (EvolvingLMMs-Lab/lmms-eval)
│   └── lmms_eval/tasks/physbench/
│       ├── physbench.yaml              ← 태스크 설정
│       └── utils.py                   ← 평가 로직
├── requirements.txt
└── README.md
```

### 모델 타입별 동작

| 모델 타입 | 사용 함수 | 비디오 처리 |
|---|---|---|
| `openai_compatible_chat` | `doc_to_messages` | 프레임 → base64 image_url |
| `vllm_chat` | `doc_to_messages` | 프레임 → base64 image_url |
| `qwen3_vl_chat` | `doc_to_messages` | `{"type": "video"}` 네이티브 처리 |
| 단순 모델 (legacy) | `doc_to_visual` + `doc_to_text` | 비디오 경로 문자열 |
