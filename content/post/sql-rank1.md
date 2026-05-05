---
author: "NoviceWyatt"
title: "최고 점수자를 구할 때 동점자가 있으면 어떻게 할까?"
date: "2026-05-05"
description: "SQL에서 최고 점수 산출 시 발생할 수 있는 동점자 처리 문제를 HAVING 절과 RANK() 함수를 통해 해결하는 방법을 탐색합니다."
tags: [
    "sql"
]
categories: [
    "sql"
]
series: ["SQL Study"]
archives: ["2026/05"]
---
SQL 공부를 위해 주기적으로 프로그래머스에서 SQL 문제를 풀고 있다.

오늘도 문제를 풀며 테스트 결과를 보기 위해 테스트 케이스를 돌리고 테스트가 패스된 김에 제출을 해보았는데 통과되었다. 그러나 나는 되려 의문에 빠졌다. 내가 생각하기에 이건 정답이 될 순 없다고 생각했기 때문이다.

내가 풀었던 문제는 이것이고, 
![Programmers SQL problem page about selecting the highest scorer with tied scores shown on a challenge page](https://school.programmers.co.kr/learn/courses/30/lessons/284527)

문제의 목적은
 > 테이블에서 2022년도 한해 평가 점수가 가장 높은 사원 정보를 조회하려 합니다 

아래는 내가 처음 제출한 코드이다.

```Mysql
SELECT 
    SUM(hg.SCORE) AS SCORE,
    he.EMP_NO,
    he.EMP_NAME,
    he.POSITION,
    he.EMAIL
FROM HR_GRADE hg  
JOIN HR_EMPLOYEES he 
    ON hg.EMP_NO = he.EMP_NO
WHERE hg.YEAR = 2022
GROUP BY hg.EMP_NO
ORDER BY SCORE DESC
LIMIT 1;
```

얼핏보기에는 원하는 결과는 출력된다고 생각한다. 그런데 이 코드에서는 문제에서 "최고 점수자가 복수인 경우는 어떻게 처리할 것인가"가 제대로 고려되지 않았다.(추가적으로 GROUP BY를 하나만 한것도 이상하긴하다;;) 

물론 문제에서 평가는 반년을 나누어 1분기, 2분기 평가를 따로 하고 있고 백점 만점 방식에 두 점수의 합으로 평가하니 확률이 그렇게 높지 않지만 그렇다고 일어나지 않을 만큼 낮은 확률도 아니라고 생각한다.

그렇다면 **"데이터에서 최고 점수자가 복수인 경우 어떻게 구해야할까?"** 라는 의문이 생기어 공부해보았다.

결론적으로 크게 두 가지 방법이 있으며 하나는 HAVING 이용한 방법 나머지는 RANK를 이용한 방법이다.

**그전에**  
리마인드 삼아 문제의 요구사항들을 정리해보자.
(생각해보니 어떤 문제인지 제대로 적어두지 않았다.😅)

1. 2022년도 평가 점수 총합이 가장 높은 사원의 정보가 필요하다,
2. 사원 정보는 점수(총합), 사번, 성명, 직책, 이메일이다.
3. 점수(총합)의 컬럼명은 SCORE로 하여라
*(주어진 테이블은 총 3개로 직원 정보가 담긴 `HR_EMPLOYEES`, 부서 정보를 담은 `HR_DEPARTMENT`, 평가 정보가 담긴 `HR_GRADE`이다.)*

여기서 점수의 총합은 HR_GRADE에서 집계하고, 사원의 정보는 HR_EMPLOYEES에서 조회하면 된다.
*(`HR_DEPARTMENT`는 문제에서 쓰이는 일은 없었다. 만약 부서도 출력하도록 했으면 같이 Join했을 것같다.)*

그렇다면 문제를 풀기위해 해야할 일은 다음과 같다.

1. `HR_GRADE`에서 2022년도 평가점수를 추려낸다.
2. 추려낸 점수를 사원별로 합산하다.
3. 최고 점수자를 찾는다.
4. 최고 점수자의 사원 정보를 `HR_EMPLOYEES`에서 조회한다.
5. 합산한 점수를 `SCORE` 컬럼으로 같이 조회한다.


3번을 제외한 모든 동작들은 기존의 코드와 같으니 빠르게 3번을 구하는 방법에 대해 이야기하자.

## 1. HAVING을 이용해 불쌍한 동점자를 찾아주자
HAVING은 서브쿼리를 이용한 집계한 결과를 비교하는 방식이다. 정리하면 그룹화후 필터링을 하기위한 것이다.
이를 이용하면 사원별로 GROUP BY가 적용된 결과에 대해 MAX()만 뽑아내면 최고 점수자가 복수더라도 모두 뽑아낼 수 있다.

```Mysql
SELECT 
    SUM(hg.SCORE) AS SCORE,
    he.EMP_NO,
    he.EMP_NAME,
    he.POSITION,
    he.EMAIL
FROM HR_GRADE hg
JOIN HR_EMPLOYEES he
    ON hg.EMP_NO = he.EMP_NO
WHERE hg.YEAR = 2022
GROUP BY 
    he.EMP_NO, 
    he.EMP_NAME, 
    he.POSITION, 
    he.EMAIL
HAVING SUM(hg.SCORE) = (
    SELECT 
        MAX(TOTAL_SCORE)
    FROM (
        SELECT SUM(SCORE) AS TOTAL_SCORE
        FROM HR_GRADE
        WHERE YEAR = 2022
        GROUP BY EMP_NO
    ) t
);
```
서브쿼리를 통해 HR_GRADE에서 최고 점수를 구해 이것과 hg.SCORE의 합과 HAVING 절을 통해 비교하도록 하면 GROUP BY절에 대해 최고 점수를 조건으로 출력할 수 있다.

## 2. RANK를 이용해 숨은 동점자를 찾아내자

이런 순위에 대한 쿼리에서 적절한 것은 서브쿼리를 이용해 복잡하게 HAVING 조건을 만드는 것보다 윈도우 함수를 이용한 방법이다.

먼저, 윈도우 함수란 행과 행 사이의 관계를 정의하기 위해 사용하는 함수이다. 집계 함수가 GROUP BY를 통해 여러 행을 하나의 결과로 합치는 것과 달리 각 행을 유지한 채로 다른 행과의 비교 연산을 할 수 있다. 그래서 결과 집합의 행 수가 줄어든거나 하지 않는다. 또한, OVER 절을 통해 어떻게 나눌지와 어떤 순서로 계산할 지를 결정한다.

이 윈도우 함수 중에 RANK함수가 있는데, 정렬된 결과 집합 내에서 각 행의 순위를 부여하는 함수이다. 값의 크기에 따라 순위를 정의할 때 사용된다. 

이때 같은 값이 있으면 RANK 역시 같기에 서브 테이블을 만들어 총합에 대해 랭크를 부여해 랭크 1만 출력하도록 하면 문제에서 원하는 결과를 출력할 수 있을 것이다

```Mysql
SELECT
    t.SCORE, 
    t.EMP_NO, 
    t.EMP_NAME, 
    t.POSITION, 
    t.EMAIL
FROM (
    SELECT
        SUM(hg.SCORE) AS SCORE,
        he.EMP_NO,
        he.EMP_NAME,
        he.POSITION,
        he.EMAIL,
        RANK() OVER (
            ORDER BY SUM(hg.SCORE) DESC
        ) AS rnk
    FROM HR_GRADE hg
    JOIN HR_EMPLOYEES he
        ON hg.EMP_NO = he.EMP_NO
    WHERE hg.YEAR = 2022
    GROUP BY 
        he.EMP_NO, 
        he.EMP_NAME, 
        he.POSITION, 
        he.EMAIL
) t
WHERE rnk = 1;
```
이 두 방식을 이용한다면 동점자가 존재하는 경우 동점자의 정보 역시 같이 조회된 결과를 볼 수 있을 것이다.s