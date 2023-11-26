# 5. Navigating Between Pages
- <a /> 태그를 사용하면 full page refresh가 일어나는 반면 <Link /> 컴포넌트는 필요한 부분만 요청하는 것을 확인할 수 있음
	![[Screen Recording 2023-11-25 at 11.43.28 PM.mov]]
- <Link /> 컴포넌트의 사용은 client-side navigation을 가능하게 해준다.
	- 어떻게 가능?: 어플리케이션 코드는 route segments에 의해 자동으로 코드가 나뉘어짐. 그리고 client에서 넥제가 route segments들을 prefetches하고 캐싱함
		- 코드가 나뉘어지면 장점은 특정 부분이 에러가 나도 다른 한쪽은 여전히 작동할 수 있다는 것
		- production에서 넥제는 <Link /> 컴포넌트가 있다는 것을 확인하면 그 즉시 prefetch를 해둠 ![[Screenshot 2023-11-26 at 12.10.51 AM.png]]
	- Client Side Navigation이란:
		- prefetch: 사용자가 방문하기 전 route를 미리 로드하는 것
			- <Link /> 컴포넌트 사용 혹은 router.prefetch()를 이용해 prefetch를 할 수 있음
			- production에서만 작동함
		- cache: 넥제는 Router Cache라는 in-memory client-side cache를 갖고 있음
			- in-memory: 디스크가 아닌 주 메모리에 모든 데이터를 보유하고 있는 데이터베이스를 뜻함
			- 즉, 사용자가 탐색 중에 서버에 새로운 요청이 발생되는 걸 줄임
		- Partial Rendering
		- Soft Navigation
			- hard navigation: page를 매번 reload (useState로 만들어진 상태와 같은 것들이 매번 리셋)
			- 하지만 넥제는 soft navigation으로 상태가 매번 업데이트되지 않도록 도와줌
		- Back and Foward Navigation
			- 자동으로 scroll position을 유지시킴

# 6. Setting Up Your Database
- route.ts 파일에 region을 설정해둘 수 있음. 이렇게 하면 latency 감소
```ts
export const runtime = 'edge'; // 'nodejs' is the default
export const preferredRegion = 'iad1'; // only execute this function on iad1
export const dynamic = 'force-dynamic'; // no caching
 
export function GET(request: Request) {
  return new Response(`I am an Edge Function!`, {
	status: 200,
  });
}
```
- 혹은 vercel project setting에서 설정할 수 있음
- database region은 한번 설정하면 못바꾸니까 신중하게 설정할 것
# 7. Fetching Data
- relational databases를 위해 SQL 혹은 ORM (Prisma 같은) 걸 쓸 수 있다
- 나중에 prisma 세팅할 때 이거 보고 하면 좋을듯 [링크](https://vercel.com/docs/storage/vercel-postgres/using-an-orm#)
- 넥제는 prerender 해버리기 때문에 Server Component를 이용한 data fetching을 한 후 변경이 일어나더라도 반영이 안된다
- Requset Waterfalls?
	- 네트워크 request가 이전 request의 종료에 의존적인 것
	```ts
	const revenue = await fetchRevenue();
	const latestInvoices = await fetchLatestInvoices(); // wait for fetchRevenue() to finish
	const {
	  numberOfInvoices,
	  numberOfCustomers,
	  totalPaidInvoices,
	  totalPendingInvoices,
	} = await fetchCardData(); // wait for fetchLatestInvoices() to finish
	```
	- Promise.all() 또는 Promise.allSettled()를 사용하면 해결할 수 있음
	- Promise.all()의 경우 하나라도 실패하면 모두 실패, 하지만 Promise.allSettled()는 그렇지 않음. 아래와 같이 보여줌.
	```js
	Promise.allSettled([
	  Promise.resolve(33),
	  new Promise((resolve) => setTimeout(() => resolve(66), 0)),
	  99,
	  Promise.reject(new Error("an error")),
	]).then((values) => console.log(values));
	
	// [
	//   { status: 'fulfilled', value: 33 },
	//   { status: 'fulfilled', value: 66 },
	//   { status: 'fulfilled', value: 99 },
	//   { status: 'rejected', reason: Error: an error }
	// ]
	```
# 8. Static and Dynamic Rendering
- Static Rendering (default)
	- routes들이 build time 혹은 data 검증 후 background에서 렌더링됨
	- 이 결과들은 캐시되어 CDN에 저장됨
	- Static Rendering은 라우터가 user에게 personalized 되지 않은 데이터를 갖고 있을 경우 유용함
	- 웹사이트 더 빨리 볼 수 있음. Server Load가 줄어들음, SEO가 최적화됨
- Dynamic Rendering
	- request time에 렌더링 됨
	- cookie나 url's search params 에 의해 정보가 결정될 때
	- real-time-data를 볼 수 있는 장점, user-specific한 data를 볼 수 있는 장점, 요청 시간 당시의 데이터를 확인할 수 있다는 장점
		```jsx
		import { unstable_noStore as noStore } from 'next/cache'; 
		export async function fetchRevenue() { 
		// Add noStore() here to prevent the response from being cached. 
		// This is equivalent to in fetch(..., {cache: 'no-store'}).
			noStore(); 
		// ...}
		```
	- 그런데, 이렇게 noStore를 다 때려버렸을 때의 문제점은 이 모든 데이터가 다 불러오기까지 기다린 다음 페이지가 나타난다는 것. 즉 초기 로딩이 엄청 느려짐
	- 다이나믹 렌더링을 할 때 이처럼 고려해야 할 점은, 가장 느린 data fetch에 맞춰서 로딩 속도가 결정된다는 것을 명심
- 단, static과 dynamic 은 route에 따라 딱 결정되는 건 아니고 spectrum 에 더 가까움
- [Caching in Next.js](https://nextjs.org/docs/app/building-your-application/caching#data-cache) 꼭 읽어보자
	- [Data Fetching...](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating) 이것두
- dynamic function 혹은 uncached data request 가 확인되면 모든 라우트에 다이나믹 렌더링을 시작한다. (하나라도 만족하지 않으면 다이나믹 렌더링 시전함)![[Screenshot 2023-11-26 at 12.42.28 PM.png]]
- ServerComponent에서 쿠키랑 헤더 혹은 search params, 혹은 page props를 사용하면 다이나믹 렌더링이 일어남
	- search params 사용하면 suspense와 함께 사용할 것을 권장 -> [이렇게 안전하게 감싸주기](https://nextjs.org/docs/app/api-reference/functions/use-search-params#static-rendering)
- Streaming
	- 점진적인 렌더링을 가능하게 해줌


# 오늘의 영단어
- commented out: 주석 처리하다
	- They are currently commented out to prevent the application from erroring.
- revalidate: 재확인하다, 재검증하다
- opt into: 선택하다
- semantics: 의미 구조
	- By default, `@vercel/postgres` doesn't set its own caching semantics.