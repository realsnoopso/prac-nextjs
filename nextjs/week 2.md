5. Navigating Between Pages
	- a 태그를 사용하면 full page refresh가 일어나는 반면 <Link /> 컴포넌트는 필요한 부분만 요청하는 것을 확인할 수 있음
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

6. Setting Up Your Database
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
	- 