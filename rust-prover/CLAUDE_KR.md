# CLAUDE.md (한국어)

이 파일은 Claude Code (claude.ai/code)가 이 저장소에서 작업할 때 참고하는 가이드입니다.

## 프로젝트 개요

이 프로젝트는 Groth16을 사용한 영지식 증명(ZKP) 시스템의 Rust 구현으로, 정확한 생년월일을 노출하지 않고 나이 인증을 증명합니다. `arkworks` 라이브러리를 사용하여 3자간 신원 증명 시나리오를 구현합니다:
- **Issuer(발급자)**: 해시된 데이터로 자격증명을 발급
- **Holder(소유자)**: 나이에 대한 영지식 증명을 생성
- **Verifier(검증자)**: 민감한 정보를 알지 못한 채 증명을 검증

## 명령어

### 빌드 및 실행
```bash
# main 함수 실행 (Hardhat 블록체인 실행 및 컨트랙트 배포 필요)
cargo run --release -- --nocapture

# 테스트 실행
cargo test --release -- --nocapture
# 또는
./run_test.sh

# 특정 테스트 실행
cargo test --release test_did_scenario -- --nocapture
cargo test --release test_sha256_preimage_witness_only -- --nocapture
cargo test --release test_sha256_preimage_public_input -- --nocapture
```

### Main 함수 실행 전 준비사항
1. `solidity-verifier/` 폴더에서 Hardhat 시작 (solidity-verifier/README.md 참고)
2. `.env` 파일에 RPC_URL과 PRIVATE_KEY 설정
3. `deploy_verifier.js`를 사용하여 컨트랙트 배포
4. main.rs:477의 `contract_address`를 배포된 주소로 업데이트

## 아키텍처

### 핵심 플로우
1. **자격증명 발급** (Issuer → Holder)
   - Issuer가 자격증명 생성: issuer_id, holder_name, holder_dob_year, randomness
   - Issuer가 모든 자격증명의 SHA256 해시를 공개 (최대: MAX_CREDENTIALS = 3)
   - 이 해시들이 검증을 위한 공개 레지스트리 역할

2. **셋업** (Verifier)
   - Verifier가 Groth16 셋업을 사용하여 proving key와 verifying key 생성
   - 더미 값을 가진 목(mock) 회로를 사용하여 범용 키 생성

3. **증명 생성** (Holder)
   - Holder가 AgeCircuit 구성:
     - Public inputs: CUTOFF_YEAR (2006)와 Issuer의 hashed_credentials
     - Witness: holder의 실제 자격증명
   - 회로는 두 가지를 증명:
     - 자격증명 해시가 Issuer가 공개한 해시 중 하나와 일치 (멤버십 증명)
     - 생년 < CUTOFF_YEAR (나이 검증)
   - Proving key를 사용하여 Groth16 증명 생성

4. **검증** (Verifier)
   - Verifying key를 사용하여 로컬에서 증명 검증
   - Solidity 컨트랙트를 통해 온체인(Ethereum)에서 검증 가능

### 주요 컴포넌트

#### AgeCircuit (src/data_structures/circuit.rs)
두 가지 제약조건을 구현하는 R1CS 회로:
- **자격증명 유효성**: SHA256(credential)이 공개된 해시 중 하나와 일치해야 함
- **나이 확인**: holder_dob_year < dob_cutoff_year (엄격한 비교 사용)

#### Credential (src/data_structures/credential.rs)
포함 내용: issuer_id, holder_name, holder_dob_year, randomness
`to_sha256()` 메서드가 회로와 동일한 순서로 해시 계산

#### Public vs Witness 데이터
- **Public Inputs** (검증자에게 공개):
  - CUTOFF_YEAR: 2006
  - hashed_credentials: 3 × 256 비트 = 768개의 field 원소 (각 비트가 하나의 F가 됨)
  - 총합: 769개의 public input (1 + 768)
- **Witness** (증명자만 알고 있는 비밀):
  - 실제 자격증명 내용 (issuer_id, holder_name, holder_dob_year, randomness)

### 중요한 구현 세부사항

#### 회로 내 SHA256 (src/main.rs:29-178)
- SHA256 해시가 **witness**일 때: Public input 불필요
- SHA256 해시가 **public input**일 때: 256개의 비트 각각이 별도의 field 원소가 됨
  - 바이트를 field 원소로 변환: Little-endian 비트 추출
  - 32바이트 해시 = 256비트 = 256개의 field 원소
  - 비트-to-field 변환 패턴은 main.rs:434-445 참고

#### Solidity 통합 (src/utils/solidity/)
- `ToSolidity` trait가 arkworks 타입을 Ethereum 호환 형식으로 변환
- Proof: 8개의 U256 값 (A.x, A.y, B.x1, B.x2, B.y1, B.y2, C.x, C.y)
- Verifying Key: alpha1, beta2, gamma2, delta2, 그리고 public input용 770개의 G1 포인트
- Public inputs: 769개의 field 원소 (cutoff year + 3개 해시에서 온 768비트)

#### 주요 타입 별칭 (src/main.rs:18-22)
```rust
type F = ark_bn254::Fr;                    // BN254 곡선의 Field
type Sha256Digest = Vec<u8>;               // 32바이트
type Groth16Proof = ...;
type Groth16ProvingKey = ...;
type Groth16VerifyingKey = ...;
```

## Groth16 신원증명 학습을 위한 분석 구조

다음 순서로 구현을 이해하세요:

### Phase 1: 데이터 흐름 이해
1. **읽기** `src/data_structures/credential.rs`
   - 자격증명이 어떻게 구조화되어 있는지
   - SHA256 해시가 어떻게 계산되는지 (필드 순서 주목)

2. **읽기** `src/entities/issuer.rs`
   - 자격증명이 어떻게 발급되는지
   - 해시된 자격증명이 어떻게 공개되는지
   - MAX_CREDENTIALS 제한 (3개)

3. **읽기** `src/entities/holder.rs`
   - Groth16::prove를 감싸는 간단한 래퍼

4. **읽기** `src/entities/verifier.rs`
   - 셋업이 proving/verifying key를 생성하는 방법
   - 키 생성을 위한 목(mock) 회로 패턴

### Phase 2: 회로 제약조건 이해
5. **읽기** `src/data_structures/circuit.rs`
   - **Public input 할당**: 27-40줄 (cutoff year + 3개의 자격증명 해시)
   - **Witness 할당**: 44-46줄 (실제 dob_year)
   - **제약조건 1** (50-79줄): 해시 preimage 증명
     - Witness 자격증명의 SHA256 계산
     - 3개의 공개 해시된 자격증명 중 하나와 일치함을 증명 (OR 로직)
   - **제약조건 2** (82줄): 나이 비교
     - `Ordering::Less`와 strict=true를 사용한 `enforce_cmp`

### Phase 3: Public Input 처리 이해
6. **읽기** `src/main.rs` 테스트 `test_sha256_preimage_witness_only` (104-133줄)
   - 해시가 witness일 때: public input 불필요
   - 제약조건 개수: ~27,000개

7. **읽기** `src/main.rs` 테스트 `test_sha256_preimage_public_input` (135-178줄)
   - 해시가 public일 때: 각 비트가 field 원소가 됨
   - 256개의 public input (비트당 하나)
   - 비트 추출 패턴 (164-173줄): **Little-endian** 비트 순서

8. **읽기** `src/main.rs` 테스트 `test_did_scenario` (180-282줄)
   - 3개의 자격증명을 가진 전체 워크플로우
   - Public input 구성 (210-232줄): cutoff year + 3개 해시
   - 두 명의 holder: 2005년생 (유효) 및 2007년생 (cutoff 2006에 대해 무효)
   - 선택적 공개(selective disclosure) 시연

### Phase 4: 메인 애플리케이션 플로우
9. **읽기** `src/main.rs` main 함수 (390-489줄)
   - **392-417줄**: Issuer가 자격증명 발급
   - **422-447줄**: Verifier 셋업 및 public input 준비
   - **450-464줄**: Holder가 증명 생성 및 로컬 검증
   - **467-488줄**: Solidity 형식으로 변환 후 블록체인에 전송

10. **읽기** `src/utils/utils.rs`
    - 바이트/field 변환을 위한 헬퍼 함수
    - `string_to_bytes`: 문자열을 Fr로 변환한 후 바이트로 변환

### Phase 5: 블록체인 통합 (선택사항)
11. **읽기** `src/main.rs` 함수 `send_tx` (285-388줄)
    - Proof/public inputs/vk가 Solidity용으로 어떻게 포맷되는지
    - Groth16Verifier 컨트랙트와의 ABI 상호작용

12. **확인** `abi.json`
    - Solidity verifier 컨트랙트 인터페이스

## ZKP 신원증명의 핵심 통찰

1. **선택적 공개**: Holder가 정확한 생년을 노출하지 않고 만 18세 이상임을 증명
2. **멤버십 증명**: OR 제약조건을 사용하여 자격증명이 3개의 해시된 자격증명 집합에 속함을 증명
3. **공개 레지스트리**: Issuer는 실제 자격증명이 아닌 해시를 공개
4. **회로 결정성**: 회로(제약조건)와 네이티브 코드(credential.rs)에서 동일한 해시 계산
5. **비트-to-Field 매핑**: SHA256 출력이 public input을 위해 256개의 field 원소가 됨
6. **셋업 단계**: 더미 값을 가진 목(mock) 회로가 범용 proving/verifying key 생성
7. **제약조건 개수**: ~27K개 제약조건 (SHA256 가젯이 대부분 차지)

## 추가 참고사항

### 회로 설계의 핵심 개념
- **R1CS (Rank-1 Constraint System)**: 모든 제약조건은 A × B = C 형태
- **SHA256 가젯**: arkworks의 사전 구현된 SHA256 회로 (~27K 제약조건)
- **비교 가젯**: `enforce_cmp`로 대소 관계를 제약조건으로 표현

### 보안 고려사항
- **Randomness**: 각 credential에 randomness 필드를 포함하여 동일한 정보라도 다른 해시 생성
- **공개 해시**: Verifier는 해시만 보고 원본 데이터를 알 수 없음
- **Zero-Knowledge**: 증명 과정에서 생년월일 정보가 전혀 노출되지 않음

### 성능 특성
- **Proving time**: ~수 초 (SHA256 가젯으로 인한 높은 제약조건 수)
- **Verification time**: ~수 밀리초 (Groth16의 상수 시간 검증)
- **Proof size**: 고정 크기 (~200 바이트, public input 제외)
