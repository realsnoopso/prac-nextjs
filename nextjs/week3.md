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
# 10. 



# 오늘의 영단어
- jarring: 어색한
- stagger: 휘청거리다 / staggered effect: 시차를 두는 효과
- 