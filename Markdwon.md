# 배열 구현 vs 연결 자료구조 구현 비교

## 0. 비교 기준이 되는 자료구조 크기

두 구현에서 노드 하나가 실제로 차지하는 크기를 `sizeof`로 직접 확인했다. (64비트 환경, gcc 기준)

| 구현 방식 | 저장하는 내용 | 노드(슬롯) 1개 크기 |
|---|---|---|
| 배열 | `char data[20]` + `int is_used` | 24 bytes |
| 연결 자료구조 | `char data[20]` + `struct node *left` + `struct node *right` | 40 bytes |

포인터 하나가 8바이트이기 때문에, 노드 하나만 놓고 보면 연결 자료구조 쪽이 배열보다 16바이트 더 크다. 여기에 `malloc`으로 힙 메모리를 할당할 때 붙는 관리 오버헤드(보통 16바이트 내외)까지 고려하면, 연결 자료구조는 노드 1개당 실질적으로 배열보다 더 많은 메모리를 쓴다.

하지만 실제 전체 메모리 사용량은 노드 1개 크기보다 **"배열 인덱스를 얼마나 촘촘하게 쓰는가"**, 즉 트리의 모양에 훨씬 크게 좌우된다.

---

## 1. 메모리 사용량 비교

배열 구현은 루트를 인덱스 1에 두고, 인덱스 `i`의 왼쪽 자식은 `2*i`, 오른쪽 자식은 `2*i+1`에 저장한다. 이 방식은 **실제 노드 개수와 상관없이, 트리에서 가장 인덱스가 큰 노드까지의 배열 공간을 전부 확보해야 한다**는 특징이 있다. 반면 연결 자료구조는 노드가 생길 때마다 딱 그만큼만 `malloc`하므로, 트리 모양과 상관없이 항상 `노드 개수 × 40바이트`만 사용한다.

### 1-1. 포화(완전) 이진트리 — 배열이 유리한 경우

포화 이진트리는 노드 수가 `n = 2^h - 1`이고, 배열 인덱스도 정확히 `1 ~ n`까지 빈틈없이 채워진다. 즉 배열에 낭비되는 칸이 하나도 없다.

| 높이 h | 노드 수 n | 배열 메모리 (n × 24B) | 연결 메모리 (n × 40B) | 배수 |
|---|---|---|---|---|
| 3 | 7 | 168 B | 280 B | 연결이 1.67배 더 큼 |
| 5 | 31 | 744 B | 1.21 KB | 연결이 1.67배 더 큼 |
| 10 | 1,023 | 23.98 KB | 39.96 KB | 연결이 1.67배 더 큼 |
| 15 | 32,767 | 767.98 KB | 1.25 MB | 연결이 1.67배 더 큼 |

포화 이진트리에서는 **배열이 항상 더 적은 메모리를 쓴다.** 인덱스 낭비가 없기 때문에 순수하게 슬롯 크기(24B) vs 노드 크기(40B) 차이만 남고, 그 비율은 항상 40/24 ≈ 1.67배로 일정하다.

### 1-2. 편향 이진트리 — 배열이 극도로 불리한 경우

한쪽으로만 자식이 계속 이어지는 편향 이진트리는 노드 수는 `n`개뿐인데, 매 레벨마다 인덱스가 2배씩 뛰어오른다. 예를 들어 왼쪽으로만 뻗은 트리는 인덱스가 `1, 2, 4, 8, ..., 2^(n-1)`로 나타나므로, 배열은 최소 `2^(n-1)`칸을 확보해야 한다. **노드 수에 대해 배열 필요 공간이 지수적으로 증가한다.**

| 노드 수 n | 배열이 필요한 슬롯 수 (2^(n-1)) | 배열 메모리 | 연결 메모리 (n × 40B) | 배열/연결 배수 |
|---|---|---|---|---|
| 5 | 16 | 384 B | 200 B | 2배 |
| 10 | 512 | 12.00 KB | 400 B | 31배 |
| 15 | 16,384 | 384.00 KB | 600 B | 655배 |
| 20 | 524,288 | 12.00 MB | 800 B | 15,729배 |
| 25 | 16,777,216 | 384.00 MB | 1,000 B | 402,653배 |

노드가 25개뿐인 편향 트리 하나를 배열로 표현하려고 384MB를 잡아먹는 반면, 연결 자료구조는 딱 1,000바이트만 쓴다. **편향 이진트리에서는 연결 자료구조가 압도적으로 유리하다.**

### 1-3. 일반(적당히 불균형한) 이진트리 — 트리 모양에 따라 달라짐

노드 20개짜리 트리라도 모양에 따라 최대 인덱스가 크게 달라질 수 있다.

| 최대 인덱스 | 배열 메모리 | 연결 메모리 (20 × 40B = 800B) |
|---|---|---|
| 25 (거의 균형) | 600 B | 800 B |
| 60 (약간 치우침) | 1.41 KB | 800 B |
| 150 (많이 치우침) | 3.52 KB | 800 B |

균형에 가까운 트리는 배열이 더 작을 수도 있고, 한쪽으로 치우칠수록 배열이 점점 불리해진다. 즉 **일반적인 트리에서는 "얼마나 균형 잡혀 있는가"가 배열 구현의 메모리 효율을 결정한다.**

### 1-4. 결론

- 트리가 완전/포화에 가까울수록 → **배열이 유리** (인덱스 낭비가 없고, 슬롯 크기도 더 작음)
- 트리가 편향에 가까울수록 → **연결 자료구조가 압도적으로 유리** (배열은 인덱스가 지수적으로 커져서 메모리가 폭증하지만, 연결 자료구조는 항상 노드 수에 정확히 비례)
- 실무에서 입력 데이터의 모양을 예측할 수 없다면, 메모리 안정성 면에서는 연결 자료구조 쪽이 훨씬 안전하다.

---

## 2. 자식 / 부모 / 형제 노드 조회 프로그램

### 2-1. 배열 구현

배열 구현은 부모-자식 관계가 **인덱스 계산만으로** 바로 나온다. 인덱스만 알고 있으면 자식/부모/형제 모두 O(1)이다.

```c
int find_index_by_value(char value[])
{
    int i;
    for (i = 1; i <= max_index; i++)
    {
        if (is_used[i] == 1 && strcmp(node_data[i], value) == 0)
        {
            return i;
        }
    }
    return -1;
}

void print_relatives(int idx)
{
    int left_idx = idx * 2;
    int right_idx = idx * 2 + 1;
    int parent_idx = idx / 2;
    int sibling_idx;

    printf("왼쪽 자식 : ");
    if (left_idx <= max_index && is_used[left_idx] == 1)
        printf("%s\n", node_data[left_idx]);
    else
        printf("없음\n");

    printf("오른쪽 자식 : ");
    if (right_idx <= max_index && is_used[right_idx] == 1)
        printf("%s\n", node_data[right_idx]);
    else
        printf("없음\n");

    printf("부모 : ");
    if (idx > 1 && is_used[parent_idx] == 1)
        printf("%s\n", node_data[parent_idx]);
    else
        printf("없음\n");

    printf("형제 : ");
    if (idx > 1)
    {
        sibling_idx = (idx % 2 == 0) ? idx + 1 : idx - 1;
        if (sibling_idx <= max_index && is_used[sibling_idx] == 1)
            printf("%s\n", node_data[sibling_idx]);
        else
            printf("없음\n");
    }
    else
    {
        printf("없음\n");
    }
}
```

값으로 노드를 찾는 `find_index_by_value`만 배열을 `1 ~ max_index`까지 순회하는 선형 탐색이고, 인덱스를 찾은 뒤부터는 곱셈/나눗셈 몇 번으로 부모·자식·형제가 전부 나온다.

### 2-2. 연결 자료구조 구현

연결 자료구조는 각 노드가 `left`, `right`만 가지고 있고 **부모를 가리키는 포인터가 없다.** 그래서 부모를 찾으려면 루트부터 다시 내려가면서 "내 자식이 target인가?"를 확인하는 별도의 탐색이 필요하다.

```c
struct node *find_node(struct node *cur, char value[])
{
    struct node *result;
    if (cur == NULL)
        return NULL;
    if (strcmp(cur->data, value) == 0)
        return cur;
    result = find_node(cur->left, value);
    if (result != NULL)
        return result;
    return find_node(cur->right, value);
}

struct node *find_parent(struct node *cur, struct node *target)
{
    struct node *result;
    if (cur == NULL || cur == target)
        return NULL;
    if (cur->left == target || cur->right == target)
        return cur;
    result = find_parent(cur->left, target);
    if (result != NULL)
        return result;
    return find_parent(cur->right, target);
}

void print_relatives(struct node *root, char value[])
{
    struct node *target = find_node(root, value);
    struct node *parent;
    struct node *sibling = NULL;

    if (target == NULL)
    {
        printf("해당 노드를 찾을 수 없습니다.\n");
        return;
    }

    printf("왼쪽 자식 : %s\n", target->left != NULL ? target->left->data : "없음");
    printf("오른쪽 자식 : %s\n", target->right != NULL ? target->right->data : "없음");

    parent = find_parent(root, target);
    printf("부모 : %s\n", parent != NULL ? parent->data : "없음");

    if (parent != NULL)
    {
        sibling = (parent->left == target) ? parent->right : parent->left;
    }
    printf("형제 : %s\n", sibling != NULL ? sibling->data : "없음");
}
```

값으로 노드를 찾는 `find_node`도 트리를 순회(O(n))해야 하고, 부모를 찾는 `find_parent`도 **또 한 번** 루트부터 순회(O(n))해야 한다. 자식은 `target->left`, `target->right`로 바로 나오지만, 부모·형제는 자식보다 훨씬 비용이 크다.

### 2-3. 동작 확인

같은 트리 `A,(B,(E,,),(F,,)),(C,,(D,(G,,),))` 에서 `B`를 조회하면 두 구현 모두 다음과 같이 동일한 결과를 낸다.

```
왼쪽 자식 : E
오른쪽 자식 : F
부모 : A
형제 : C
```

### 2-4. 효율성 분석

| 연산 | 배열 구현 | 연결 자료구조 구현 |
|---|---|---|
| 값으로 노드 찾기 | O(max_index) — 편향 트리에서는 노드 수 n보다 훨씬 큰 범위(최악 O(2ⁿ))를 훑어야 함 | O(n) — 항상 실제 노드 수만큼만 순회 |
| 자식 찾기 | O(1) (인덱스 `2*idx`, `2*idx+1`) | O(1) (포인터 `target->left`, `target->right`) |
| 부모 찾기 | O(1) (인덱스 `idx/2`) | O(n) (루트부터 다시 탐색해야 함, 부모 포인터가 없으므로) |
| 형제 찾기 | O(1) (인덱스 짝/홀 판별) | O(부모 찾기 비용과 동일, 이후 O(1)) |

**결론:**

- 부모/형제를 찾는 연산만 놓고 보면 **배열 구현이 압도적으로 효율적**이다. 인덱스 하나로 부모가 바로 계산되기 때문에, 연결 자료구조처럼 트리를 처음부터 다시 훑을 필요가 없다.
- 다만 트리가 편향되어 있으면 얘기가 달라진다. 배열은 "값으로 노드를 찾는" 첫 단계부터 `max_index`(최악의 경우 2ⁿ)까지 훑어야 하므로, 1절에서 본 메모리 폭증 문제와 똑같은 이유로 탐색 속도도 함께 나빠진다. 반면 연결 자료구조는 항상 노드 수 n에 비례해서만 움직인다.
- 연결 자료구조에 부모 찾기가 느린 것은 **구조상의 한계**이지 알고리즘의 한계는 아니다. 각 노드에 `parent` 포인터 필드를 하나 추가하면 부모/형제 조회도 O(1)로 만들 수 있다. 다만 그러면 노드 크기가 40바이트에서 48바이트로 늘어나서, 1절의 메모리 비교 결과가 그만큼 더 불리해진다 — 즉 **속도와 메모리는 트레이드오프 관계**에 있다.

### 2-5. 종합 결론

- 트리가 완전/포화에 가깝고 형태가 안정적으로 유지된다면 → **배열 구현**이 메모리와 속도 모두에서 유리하다.
- 트리가 한쪽으로 치우치거나(편향), 크기를 예측하기 어렵다면 → **연결 자료구조**가 메모리 낭비 없이 안전하고, 부모 조회가 자주 필요하면 `parent` 포인터를 추가해서 보완할 수 있다.
- 결국 두 구현의 우열은 고정되어 있지 않고, **입력으로 들어오는 트리의 모양**에 따라 갈린다.