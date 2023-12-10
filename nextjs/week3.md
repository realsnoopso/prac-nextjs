# 9. Streaming

- 아래와 같은 구조로 갈 경우 해당 폴더의 전체에 적용되게 됨 (다른 페이지가 없으므로 이쪽으로 넘어오는 것)
	- dashboard/(...)/page.tsx
	- dashboard/(...)/layout.tsx
- loading.tsx를 통해 페이지 전체의 fallback을 설정할 수도 있지만, 컴포넌트를 Suspense로 감싸서 일부만 fallback을 설정하는 것도 가능함. 단, 내부에 await 가 있어야 함 -> 이런거 보면 react query 걷어내야 되나 싶기도...
	```jsx
	<Suspense fallback={<RevenueChartSkeleton />}>
		<RevenueChart />
	</Suspense>


...


export default async function LatestInvoices({}: {}) {
	const latestInvoices = await fetchLatestInvoices();
	...
	```
# 10. Partial Prerendering
- next 14에만 적용되는 실험적인 피처임
- 특징
	- Fast edge delivery
	- Does not block TTFB on cold start
		- ttfb: Time to first byte 웹서버의 응답에 대한 매트릭
	- Streaming dynamic delivery
	- No extra round trips for dynamic content
		- round trip time: 인터넷 상에서 송신지부터 목적지까지 패킷이 왕복하는데 걸리는 시간
	- Fast access to core region databases and APIs
	- One simple programming model between statics and dynamics parts
- 역사 나중에 공부해보면 좋을 것 같은 키워드
	- https://en.wikipedia.org/wiki/Server_Side_Includes
	- Edge SSR
- Server component의 장점
	- This allows you to keep expensive data fetches and logic on the server, reduces the client-side JavaScript bundle, and prevents your database secrets from being exposed to the client.
	- 외부에 database secrets를 노출시키지 않아도 되고, client-side JS 번들 사이즈를 줄일 수도 있음

# 11. Adding Search and Pagination
- 여기 예시에서는 seach state를 관리하기 위해 상태를 URL search params로 관리했음 (클라이언트 사이드 렌더링이였음 useState를 썼겠지?)
	- 위와 같이 했을시 장점은 다음과 같음
		- URL을 북마킹, 공유 가능해짐
		- SSR이 가능해짐, 즉 initial load가 가능해짐
		- 분석 및 트래킹이 가능해짐
- **`defaultValue` vs. `value` / Controlled vs. Uncontrolled**
	- native input이 state를 관리하는 property가 defaultValue
- server page comonent에서는 searchParams를 prop으로 읽어올 수 있음

# 12. Mutating Data
- server action을 사용함으로서 많은 보안 취약점을 극복할 수 있음
- form element에서 server action 사용법
	- form data를 받을 수 있음
	- js가 disable 된 상태에서도 작동함
```jsx
// Server Component
export default function Page() {
  // Action
  async function create(formData: FormData) {
    'use server';
 
    // Logic to mutate data...
  }
 
  // Invoke the action using the "action" attribute
  return <form action={create}>...</form>;
}
```
- 서버 액션은 또한 nextjs 캐싱과도 잘 결합되어 있음
	- `revalidatePath`, `revalidateTag`와 같은 api들을 사용
- form에 post api를 바로 붙일 수 있음
```jsx
<form action={createInvoice}>
</form
```

- zod에서 string을 넘버로 자동으로 변환해주는 옵션 있음
	- amount: z.coerce.number()
- revalidatePath: prefetching이 일어난 path들이 있는데, 캐시를 무시하고자 할 때 사용 가능
```jsx
'use server';
 
import { z } from 'zod';
import { sql } from '@vercel/postgres';
import { revalidatePath } from 'next/cache';
import { redirect } from 'next/navigation';
 
// ...
 
export async function createInvoice(formData: FormData) {
  const { customerId, amount, status } = CreateInvoice.parse({
    customerId: formData.get('customerId'),
    amount: formData.get('amount'),
    status: formData.get('status'),
  });
  const amountInCents = amount * 100;
  const date = new Date().toISOString().split('T')[0];
 
  await sql`
    INSERT INTO invoices (customer_id, amount, status, date)
    VALUES (${customerId}, ${amountInCents}, ${status}, ${date})
  `;
 
  revalidatePath('/dashboard/invoices');
  redirect('/dashboard/invoices');
}
```

# 오늘의 영단어
- jarring: 어색한
- stagger: 휘청거리다 / staggered effect: 시차를 두는 효과
- vulnerable: 상하기 쉬운, 취약한
- versatile: 융통성 있는
- 