## Type Cast Operator

static_cast<변환할타입>(변환대상)
→ 타입변환을 하는 경우, 논리적으로 가능한 경우에만 casting을 허용한다. 

const_cast<변환할타입>(변환대상)
→ 포인터 또는 참조형에서만 사용이 가능하며, const 또는 volatile값을 임시적으로 제거할 때 사용한다. 

reinterpret_cast<변환할타입>(변환대상)
→ 명시적 변환과 동일한 동작을 지원한다. (const를 사용하는 변수에는 사용이 불가능하다)

dynamic_cast<변환할타입>(변환대상)
→ Class의 포인터간 형변환을 하는 경우, 안전하게 down casting을 하기 위해 사용한다.
