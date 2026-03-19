# neo::regedit - C++ Windows Registry Library

## 개요

`regedit.hpp`는 Windows 레지스트리를 C++ STL 스타일로 다룰 수 있게 해주는 헤더 전용(header-only) 라이브러리입니다.
`std::map`과 유사한 인터페이스를 제공하여, 레지스트리 키와 값을 직관적으로 탐색하고 조작할 수 있습니다.

- **작성자**: neo3587
- **라이선스**: 저장소 내 LICENSE 파일 참고
- **최소 요구 사항**: C++11 이상, Windows 환경

---

## 주요 특징

- **헤더 전용**: `regedit.hpp` 하나만 포함하면 바로 사용 가능
- **STL 호환 인터페이스**: `std::map`과 유사한 API (iterator, `[]`, `at`, `insert`, `erase`, `find`, `clear`, `emplace` 등)
- **양방향 이터레이터**: `iterator` / `const_iterator` / `reverse_iterator` / `const_reverse_iterator` 지원
- **서브키 및 값 분리 관리**: `neo::regedit`(서브키)와 `neo::regedit::values`(값) 객체를 통해 독립적으로 접근
- **모든 레지스트리 값 타입 지원**: 아래 표 참고
- **유니코드 쓰기 지원**: `write_unicode()` 메서드로 와이드 문자열 기반 쓰기 가능
- **읽기/쓰기 권한 제어**: 키 열기 시 `write_permission` 인자로 읽기 전용 모드 선택 가능

---

## 지원하는 레지스트리 값 타입

| 타입 열거자 | Windows 타입 | C++ 반환 타입 |
|---|---|---|
| `type::none` | `REG_NONE` | `void*` |
| `type::sz` | `REG_SZ` | `std::string` |
| `type::expand_sz` | `REG_EXPAND_SZ` | `std::string` (환경변수 확장) |
| `type::binary` | `REG_BINARY` | `std::unique_ptr<BYTE>` |
| `type::dword` | `REG_DWORD` | `DWORD` |
| `type::dword_big_endian` | `REG_DWORD_BIG_ENDIAN` | `DWORD` |
| `type::link` | `REG_LINK` | `std::wstring` |
| `type::multi_sz` | `REG_MULTI_SZ` | `std::vector<std::string>` |
| `type::resource_list` | `REG_RESOURCE_LIST` | `std::unique_ptr<BYTE>` |
| `type::full_resource_descriptor` | `REG_FULL_RESOURCE_DESCRIPTOR` | `std::unique_ptr<BYTE>` |
| `type::resource_requirements_list` | `REG_RESOURCE_REQUIREMENTS_LIST` | `std::unique_ptr<BYTE>` |
| `type::qword` | `REG_QWORD` | `DWORD64` |

---

## 클래스 구조

```
neo::regedit                  ← 레지스트리 키를 나타내는 메인 클래스
├── neo::regedit::hkey        ← 루트 키 상수 (HKEY_*)
├── neo::regedit::type        ← 값 타입 열거형
├── neo::regedit::value       ← 개별 레지스트리 값
├── neo::regedit::values      ← 키 내 값 집합 (map-like)
├── neo::regedit::iterator
└── neo::regedit::const_iterator
```

### 루트 키 상수 (`neo::regedit::hkey`)

| 상수 | Windows 매크로 |
|---|---|
| `hkey::classes_root` | `HKEY_CLASSES_ROOT` |
| `hkey::current_config` | `HKEY_CURRENT_CONFIG` |
| `hkey::current_user` | `HKEY_CURRENT_USER` |
| `hkey::local_machine` | `HKEY_LOCAL_MACHINE` |
| `hkey::users` | `HKEY_USERS` |

---

## 주요 API

### 키 열기 / 닫기

```cpp
neo::regedit reg(neo::regedit::hkey::current_user, "Software\\MyApp", true);
reg.open(neo::regedit::hkey::local_machine, "SOFTWARE", false); // 읽기 전용
reg.close();
bool opened = reg.is_open();
```

### 서브키 접근

```cpp
reg["SubKey"];           // 없으면 생성
reg.at("SubKey");        // 없으면 std::out_of_range 예외
reg[0];                  // 인덱스로 접근
reg.find("SubKey");      // iterator 반환, 없으면 end()
```

### 서브키 추가 / 삭제

```cpp
reg.insert("NewKey");
reg.erase("OldKey");
reg.clear();             // 모든 서브키 삭제
```

### 값(value) 접근 및 읽기/쓰기

```cpp
// 쓰기
reg.values["my_string"].write<neo::regedit::type::sz>("hello");
reg.values["my_dword"].write<neo::regedit::type::dword>(42u);

// 읽기
std::string s = reg.values["my_string"].read<neo::regedit::type::sz>();
DWORD d       = reg.values["my_dword"].read<neo::regedit::type::dword>();

// 타입 및 크기 조회
neo::regedit::type ty = reg.values["my_string"].type();
size_t sz             = reg.values["my_string"].size();

// 값 삭제
reg.values.erase("my_string");
```

### 이터레이터 순회

```cpp
// 서브키 순회
for (auto it = reg.begin(); it != reg.end(); ++it)
    std::cout << it->first << "\n"; // 키 이름

// 값 순회
for (auto it = reg.values.begin(); it != reg.values.end(); ++it)
    std::cout << it->first << ": "
              << neo::regedit::type_to_string(it->second.type()) << "\n";
```

---

## 사용 예시

```cpp
#include "regedit.hpp"
#include <iostream>

int main() {
    neo::regedit reg(neo::regedit::hkey::current_user, "Software", true);
    reg = reg["_test_key"];

    // 서브키 10개 추가
    for (int i = 0; i < 10; i++)
        reg.insert("key " + std::to_string(i));

    // 값 쓰기
    reg.values["sz_val"].write<neo::regedit::type::sz>("hello");
    reg.values["dw_val"].write<neo::regedit::type::dword>(12345u);

    // 서브키 목록 출력
    std::cout << "Subkeys: " << reg.size() << "\n";
    for (auto it = reg.begin(); it != reg.end(); ++it)
        std::cout << "  " << it->first << "\n";

    // 값 목록 출력
    std::cout << "Values: " << reg.values.size() << "\n";
    for (auto it = reg.values.begin(); it != reg.values.end(); ++it)
        std::cout << "  " << it->first
                  << " (" << neo::regedit::type_to_string(it->second.type()) << ")\n";

    return 0;
}
```

---

## 주의 사항

- **Windows 전용**: `<windows.h>`에 의존하므로 Windows 환경에서만 동작합니다.
- **레지스트리 가상화**: 일부 키는 레지스트리 가상화로 인해 다른 경로로 리디렉션될 수 있습니다.
  참고: [Registry Virtualization (MSDN)](https://docs.microsoft.com/es-es/windows/desktop/SysInfo/registry-virtualization)
- **쓰기 권한**: 일부 키는 쓰기 권한으로 열 수 없습니다.
- **WDK 타입**: `resource_list`, `full_resource_descriptor`, `resource_requirements_list` 타입은 WDK의 `wdm.h`에 정의된 구조체로 캐스팅이 필요합니다.
- **백업 권장**: 레지스트리를 수정하기 전에 반드시 백업하세요.
  참고: [레지스트리 백업 방법 (Microsoft 지원)](https://support.microsoft.com/en-us/help/322756)
