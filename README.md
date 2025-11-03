# CODIIT-backend 프로젝트 개인 개발 보고서

**프로젝트명:** CODI-IT  
**프로젝트 기간:** 2025.09.15 ~ 2025.11.03  
**팀명:** 2팀  
**작성자:** 진성남

---

## 🧩 1. 개요

본 프로젝트는 **패션 쇼핑몰 플랫폼 CODIIT**의 백엔드 서버를 구축하기 위한 협업 프로젝트입니다.  
프로젝트의 핵심 목표는 **안정적이고 확장 가능한 백엔드 아키텍처**를 구현하는 것이었으며,  
저는 **상품(Product)** 과 **주문(Order)** 기능 개발을 중심으로 맡았습니다.

서비스는 **NestJS + PrismaORM + PostgreSQL** 기반으로 구현되었으며,  
**AWS 인프라(EC2, RDS, S3)** 에 배포하여 실제 프로덕션 환경 수준의 아키텍처를 구성했습니다.

---

## ⚙️ 2. 기술 스택 및 협업 도구

| 분류 | 사용 도구 |
|------|------------|
| **Backend** | Node.js (Express → NestJS) + TypeScript |
| **Database** | PostgreSQL + PrismaORM + AWS RDS + AWS S3 |
| **API 문서화** | Swagger |
| **협업 도구** | Discord, GitHub, Notion |
| **일정 관리** | GitHub Issues + Notion 타임라인 |
| **테스트** | Jest + SuperTest |
| **배포** | AWS EC2 + Nginx + PM2 |
| **CI/CD** | GitHub Actions |

---

## 🏗️ 3. 프로젝트 아키텍처

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/f6016251-f7ef-4423-b9b5-71ca3e422838" />


> 프론트엔드(Vercel)와 백엔드(NestJS) 간 Axios 통신,  
> 백엔드는 EC2 상에서 Docker + Nginx + PM2 환경으로 동작하며  
> 데이터는 AWS RDS(PostgreSQL)와 S3를 통해 관리됩니다.

---

## 🧱 4. 담당 기능 요약

| 구분 | 주요 구현 내용 |
|------|----------------|
| **상품(Product)** | 상품 등록/수정/삭제/조회 기능 구현<br>할인율 계산 로직 적용<br>사이즈 자동 생성 로직 추가<br>리뷰/문의 관계 데이터 조회 포함 |
| **주문(Order)** | 주문 생성/조회/취소 기능 구현<br>상품 및 재고 연동 처리<br>포인트 차감 및 누적 로직 연동<br>트랜잭션 기반 주문 데이터 무결성 보장 |
| **공통 구조** | Prisma 기반 Repository 레이어 구축<br>Service 레이어에서 DTO 검증 및 예외 처리<br>Swagger 문서 자동화 적용 |
| **에러 처리** | 커스텀 예외 클래스 적용 및 전역 필터 구성 |
| **테스트** | Jest + SuperTest 기반 e2e 테스트 작성 |

---


## 주요 개발 
### 5 상품·주문 도메인 주요 기능 개발 요약

---

#### 🛍️ **상품(Product) 도메인 개발**

**핵심 목표:**  
상품 정보, 가격, 재고를 안정적으로 관리하고 프론트엔드 요구사항에 맞춘 응답 구조를 설계.

**주요 구현 내용:**

1. **상품 등록 / 수정**
   - **할인 로직 자동 계산**  
     `discountStartTime`, `discountEndTime`을 비교해 `isDiscounted`(“Y”/“N”) 판정,  
     할인 중이면 `discountPrice`를 실시간 계산하여 응답에 포함.
   - **사이즈 자동 생성 로직**  
     `sizeId` 미존재 시 `createStockSize()`로 신규 사이즈를 자동 생성.  
     → Seed 없이도 상품 등록 가능.
   - **카테고리 자동 생성**  
     프론트에서 전달한 카테고리명이 DB에 없으면 자동 생성 처리.
   - **판매자 권한 검증**  
     `sellerId`와 상품의 `storeId`를 비교하여 불일치 시 `ForbiddenException` 발생.

2. **상품 목록 / 상세 조회**
   - **ProductListResponse DTO 설계**  
     프론트 요구사항에 맞게 `discountRate`, `reviewsCount`, `reviewsRating` 필드 포함.  
     불필요한 내부 데이터 제외하여 효율적 응답 제공.
   - **관계 데이터 포함 조회**  
     `store`, `category`, `stocks`, `reviews`, `inquiries`를 Prisma `include`로 한번에 조회.
   - **할인 여부 계산 자동화**  
     FE에서 계산 로직 불필요, 단순 렌더링만 가능하도록 설계.

3. **상품 문의(Inquiry)**
   - 상품 존재 검증 후, `CreateInquiryDto`로 문의 등록 처리.
   - `AnswerStatus`(`WAITING`, `ANSWERED`)로 문의 상태를 명확히 구분.

**결과:**  
- 상품 등록 시 필수 관계 데이터 누락 문제 해소.  
- 프론트엔드에서 바로 사용할 수 있는 응답 구조 완성.  
- 판매자/구매자 권한 로직 명확화로 운영 안정성 확보.

---

#### 📦 **주문(Order) 도메인 개발**

**핵심 목표:**  
주문 생성부터 재고 차감, 포인트 사용까지를 **하나의 트랜잭션으로 처리**하여 데이터 무결성 확보.

**주요 구현 내용:**

1. **주문 생성 (`POST /api/orders`)**
   - Prisma `$transaction()`으로  
     `Order`, `OrderItem`, `Stock`, `PointTransaction`을 **원자적 연산으로 처리**.  
   - 주문 시 상품 재고 차감(`quantity - 주문수량`) 자동 반영.  
   - 포인트 사용/적립 로직을 동시에 반영 (`decrement`, `increment`).

2. **주문 조회 / 상세 / 취소**
   - **JWT 인증 기반 사용자 검증**  
     로그인한 사용자만 자신의 주문 내역 조회 가능.
   - **관계 데이터 통합 조회**  
     `OrderItem`, `Product`, `Stock`을 Prisma Relation으로 함께 반환.  
   - **주문 취소 로직 개선**  
     `OrderStatus` → `CANCELED` 전환 시 포인트 환불 트랜잭션 적용.

3. **주문 통계 (Dashboard 연동)**
   - Prisma `groupBy`를 활용해 월별 주문 수, 총 매출, 평균 단가 계산.  
   - 관리자 대시보드용 통계 API로 제공.

**결과:**  
- 주문과 재고의 불일치 문제 해소.  
- 트랜잭션 기반 설계로 동시 주문 상황에서도 안정성 확보.  
- FE에서 주문 상태 관리 및 통계 화면 구현이 간소화됨.

---

#### ⚙️ **기술적 설계 포인트**

- **Layered Architecture** 적용 (`Controller → Service → Repository`)  
- **DTO + ValidationPipe**로 요청 데이터 유효성 강화  
- **Prisma Repository 분리**로 테스트 독립성 확보  
- **Custom ExceptionFilter**로 예외 응답 일관화  
- **Swagger 자동 문서화**로 프론트 협업 효율 향상

---


## 💡 6. 핵심 구현 코드 예시

### ✅ 주문 생성 
```
async createOrder(userId: string, dto: CreateOrderDto) {
  return this.prisma.$transaction(async (tx) => {
    const order = await tx.order.create({
      data: {
        buyerId: userId,
        totalAmount: dto.totalAmount,
        status: 'PAID',
      },
    });

    // 주문 상품 처리
    for (const item of dto.items) {
      await tx.orderItem.create({
        data: { orderId: order.id, productId: item.productId, quantity: item.quantity },
      });

      // 재고 차감
      await tx.stock.update({
        where: { id: item.stockId },
        data: { quantity: { decrement: item.quantity } },
      });
    }


    return order;
  });
}
```

###  ✅ 상품 목록 조회 (할인·리뷰 포함) 

```
async findProducts(): Promise<ProductListResponse> {
  const products = await this.prisma.product.findMany({
    include: { store: true, reviews: true },
    orderBy: { createdAt: 'desc' },
  });

  return products.map((p) => {
    const isDiscounted =
      p.discountStartTime &&
      p.discountEndTime &&
      new Date() >= p.discountStartTime &&
      new Date() <= p.discountEndTime;

    const reviewsCount = p.reviews.length;
    const reviewsRating =
      reviewsCount > 0
        ? Number(
            (p.reviews.reduce((acc, r) => acc + r.rating, 0) / reviewsCount).toFixed(1),
          )
        : 0;

    return {
      ...p,
      discountPrice: isDiscounted
        ? Math.floor(p.price * (1 - (p.discountRate ?? 0) / 100))
        : p.price,
      reviewsCount,
      reviewsRating,
      isDiscounted: isDiscounted ? 'Y' : 'N',
    };
  });
}
```
### 7) 프로젝트 회고 및 느낀 점

---


이번 CODIIT 프로젝트를 진행하면서 **NestJS**를 처음 활용해 보았는데,  
기존에 익숙했던 **Express**와 달라 초반에는 어려움이 있었습니다.  
하지만 하나씩 배우면서, NestJS의 **의존성 주입(DI)** 과 **인젝터블 기반 구조**를 이해하게 되었고,  
이로 인해 코드의 **유지보수성과 확장성**을 크게 높일 수 있었습니다.

또한 **예외 처리 시스템(Exception Filter)** 을 통해  
`throw`만으로 상태 코드와 응답을 자동 반환할 수 있다는 점이 인상 깊었습니다.  
이 기능 덕분에 코드가 더욱 간결해지고,  
**데코레이터 기반 문법**을 활용한 **Swagger 연동**도 훨씬 수월해졌습니다.

---

#### 🧪 테스트 경험

테스트 측면에서는 **E2E 테스트**와 **유닛 테스트**의 차이를 직접 체감했습니다.

- **상품(Product)** 과정에서는 실제 서버를 구동하여 **E2E 테스트**로  
  API 요청과 응답 흐름을 검증했습니다.  
- **주문(Order)** 과정에서는 **유닛 테스트**를 통해  
  컨트롤러와 서비스 로직을 독립적으로 테스트했습니다.

이를 통해 **E2E 테스트는 전체 서비스의 동작 검증에 유용**하며,  
**유닛 테스트는 개별 로직의 정확성과 안정성 확인에 효과적**이라는 점을 배웠습니다.  
테스트를 많이 수행할수록 코드의 **신뢰성, 보안성, 성능**이 향상된다는 것도 체감했습니다.

---

#### 🤝 협업 경험

이번 프로젝트를 통해 느낀 가장 큰 점은  
> “프로젝트는 혼자 하는 것이 아니라, 팀원들과의 소통이 핵심이다.”  
라는 것입니다.

팀에서는 **GitHub 브랜치 전략**을 적용하여  
각 작업마다 `task-번호`를 부여하고,  
이슈·PR 단위로 체계적으로 관리했습니다.  
또한 매일 **데일리 스크럼 회의**를 통해 진행 상황을 공유하며  
문제나 일정 이슈를 빠르게 조율했습니다.

처음에는 **상품·주문·알림 기능**을 모두 맡았지만,  
팀원 **진솔님**과의 역할 조율을 통해  
알림 기능은 협업으로 진행하게 되었습니다.  
비록 모든 기능을 혼자 완성하지는 못했지만,  
서로 도와가며 완성도를 높이는 협업의 진정한 의미를 배웠습니다.

---

**결론적으로**,  
이번 프로젝트는 단순한 개발을 넘어  
> “협업, 구조적 설계, 테스트, 그리고 커뮤니케이션의 중요성”  
을 깊이 이해할 수 있는 계기가 되었습니다.  
앞으로도 이러한 경험을 바탕으로  
더 견고하고 협업 친화적인 백엔드 개발자로 성장하겠습니다.

