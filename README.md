# TechDemo Real-Time Fur Rendering

Unreal Engine에서 **Shell Texturing** 기법을 활용해 실시간 Fur Rendering을 구현한 기술 데모입니다.

본 프로젝트는 완성형 게임이 아니라,
Static Mesh에서 검증한 Shell Rendering 구조를 Skeletal Mesh에 확장 적용하고,
그 과정에서 발생하는 파이프라인 제약과 성능 한계를 확인하기 위한 **Rendering Tech Demo**입니다.

---

## Project Info

| 항목               | 내용                          |
| ---------------- | --------------------------- |
| Engine           | Unreal Engine 5             |
| Implementation   | Blueprint / Material Graph  |
| Rendering Method | Shell Texturing             |
| Target Mesh      | Static Mesh / Skeletal Mesh |
| Development Type | Solo Tech Demo              |
| Period           | 2026.05 ~ 2026.06           |

---

## Overview

Shell Texturing은 동일한 Mesh를 여러 겹으로 렌더링하고,
각 Layer를 Vertex Normal 방향으로 조금씩 확장하여 털, 잔디, 솜털 같은 표면 디테일을 표현하는 방식입니다.

본 데모에서는 먼저 Static Mesh 기반으로 Shell Layer 구조를 검증한 뒤,
이를 Skeletal Mesh에 적용하는 과정을 진행했습니다.

Static Mesh에서는 `Instanced Static Mesh`와 `PerInstanceCustomData`를 이용해 각 Shell Layer의 값을 Material에 전달했습니다.

하지만 Skeletal Mesh에서는 동일한 Instancing 구조를 그대로 적용하기 어렵다고 판단했고,
Blueprint 기반의 `Multi-Layer SkeletalMeshComponent` 구조로 우회 구현했습니다.

---

## Implementation Decision

Unreal Engine의 `InstancedSkinnedMeshComponent`를 검토했지만,
해당 구조는 다수의 단순 Skeletal Mesh 인스턴스를 효율적으로 처리하는 데 초점이 있는 구조로 보았습니다.

본 데모에서는 군중 렌더링보다 다음 요소들이 더 중요했습니다.

* Shell Layer별 WPO 제어
* ShellIndex 기반 LayerRatio 계산
* Opacity Mask 기반 털 밀도 제어
* Growth Mask 기반 털 생성 영역 제한
* Skeletal Mesh 애니메이션 Pose 동기화

따라서 `SkeletalMeshComponent`를 Layer 수만큼 생성하고,
`Leader Pose Component`를 통해 SourceMesh와 동기화하는 방식을 선택했습니다.

---

## Rendering Structure

Skeletal Mesh 버전에서는 하나의 SourceMesh를 기준으로 여러 Shell Component를 생성합니다.

```text
BP_ShellFur_Skeletal
├─ SourceMesh
├─ ShellMesh_00
├─ ShellMesh_01
├─ ShellMesh_02
└─ ShellMesh_N
```

각 Shell Component는 다음 과정을 통해 초기화됩니다.

```text
Add SkeletalMeshComponent
→ Attach To SourceMesh
→ Set Skeletal Mesh Asset
→ Set Leader Pose Component
→ Disable Collision
→ Disable Shadow
→ Create Dynamic Material Instance
→ Set ShellIndex / ShellCount
```

`SourceMesh`는 실제 Skeletal Mesh와 Animation을 담당하고,
각 Shell Component는 `Leader Pose Component`를 통해 동일한 애니메이션 Pose를 공유합니다.

---

## Material Logic

Material Graph에서는 각 Shell Layer의 위치와 투명도를 계산합니다.

### Layer Ratio

각 Shell이 안쪽 Layer인지, 바깥쪽 Layer인지 판단하기 위해 `LayerRatio`를 계산합니다.

```text
LayerRatio = ShellIndex / max(ShellCount - 1, 1)
```

### World Position Offset

각 Shell Layer는 Vertex Normal 방향으로 확장됩니다.

```text
WPO = VertexNormalWS × ShellLength × LayerRatio
```

이를 통해 동일한 Skeletal Mesh가 Layer별로 조금씩 바깥쪽으로 퍼지며,
털의 부피감을 형성합니다.

### Opacity / Growth Mask

Opacity Mask는 털의 밀도를 제어하는 데 사용했습니다.

Growth Mask는 털이 생성될 영역을 제한하기 위한 흑백 텍스처입니다.

```text
Black = Fur grows
White = Fur suppressed
```

Material에서는 Growth Mask 값을 반전하여 사용했습니다.

```text
GrowthMask = 1 - TextureSample.R
```

이 값을 Opacity Mask와 WPO에 결합하여,
특정 부위에서는 털이 생성되지 않도록 처리했습니다.

---

## Features

* Shell Texturing 기반 Fur Rendering
* Static Mesh Instancing 기반 Shell Layer 검증
* SkeletalMeshComponent 기반 Multi-Layer Shell 구조
* Leader Pose Component 기반 Skeletal Mesh Pose 동기화
* Dynamic Material Instance 기반 ShellIndex 전달
* WPO 기반 Shell 확장
* Opacity Mask 기반 털 밀도 제어
* Growth Mask 기반 부위별 털 생성 제한
* ShellCount별 품질 및 성능 변화 확인

---

## Performance Notes

Skeletal Mesh 버전은 Blueprint만으로 Shell Texturing을 적용할 수 있다는 장점이 있지만,
ShellCount만큼 `SkeletalMeshComponent`가 생성되므로 렌더링 비용이 증가합니다.

```text
Rendering Cost ≈ ShellCount × Material Slot Count × Render Pass Count
```

특히 다음 비용이 증가합니다.

* SkeletalMeshComponent 수 증가
* Draw Call 증가
* Masked Material Overdraw 증가
* Shell Layer 수에 따른 렌더링 부하 증가

본 데모에서는 128 Layer까지 테스트했지만,
실시간 데모 기준으로는 16~32 Layer 수준이 적절하다고 판단했습니다.

| Shell Count | 용도                         |
| ----------: | -------------------------- |
|           8 | Low Quality / Far Distance |
|          16 | Basic Quality              |
|          32 | Recommended Demo Quality   |
|          64 | High Quality Preview       |
|         128 | Stress Test                |

---

## Applied Optimization

본 데모에서는 Blueprint 기반 프로토타입 수준에서 다음 최적화를 적용했습니다.

* Shell Component Collision 비활성화
* Shell Component Shadow 비활성화
* Material Slot 최소화 방향으로 구성
* 실시간 데모 기준 ShellCount 16~32 권장

---

## Limitations

현재 구조는 구현이 단순하고 Blueprint만으로 재현 가능하다는 장점이 있지만,
실사용 기준에서는 다음과 같은 한계가 있습니다.

* ShellCount 증가에 따른 Draw Call 증가
* SkeletalMeshComponent 복제 방식의 구조적 비용
* Masked Material Overdraw 비용
* 다수 캐릭터에 적용하기 어려움
* 상용 Fur Rendering 시스템 대비 제한적인 최적화

따라서 실제 게임 적용을 위해서는 거리 기반 Shell LOD, Component Pooling,
Custom Primitive Data 기반 파라미터 전달, 엔진 레벨 렌더링 최적화 등을 추가로 검토해야 합니다.

---

## Images

```text
Images/
├─ preview.png
├─ beautyshot.png
├─ shell_component_generation.png
├─ material_layerratio_wpo.png
├─ material_growth_mask.png
└─ shell_count_test.png
```

---

## Asset Note

본 저장소에는 외부 에셋 또는 라이선스가 필요한 Skeletal Mesh가 포함되지 않을 수 있습니다.

```text
The sample mesh used in the demo may not be included due to asset licensing.
The material and blueprint setup can be applied to other compatible Skeletal Mesh assets.
```

---

## Reference

* Unreal Engine `InstancedSkinnedMeshComponent` 검토
* Shell Texturing 기반 Fur Rendering 개념
* Unreal Engine Material Graph / World Position Offset
* Unreal Engine Blueprint `Leader Pose Component`

---

## Summary

이 프로젝트는 Unreal Engine에서 Shell Texturing 기반 Fur Rendering을 구현한 기술 데모입니다.

Static Mesh에서 검증한 Shell Rendering 구조를 Skeletal Mesh에 확장하는 과정에서,
Skeletal Mesh 파이프라인의 제약을 확인했고,
Blueprint 기반 Multi-Layer SkeletalMeshComponent 구조로 우회 구현했습니다.

이를 통해 Shell Layer별 WPO, Opacity Mask, Growth Mask 제어를 적용하고,
ShellCount 증가에 따른 품질과 성능 한계를 확인했습니다.
