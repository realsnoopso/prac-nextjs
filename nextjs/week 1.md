# 내용 정리
1. [route segment config](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config) : page, layout, route handler 등을 라우트 별로 설정할 수 있는 기능
2. [clsx](https://www.npmjs.com/package/clsx): optional 하게 class를 적용할 수 있게 돕는 libarary
3. [CLS (Cumulative Layout Shift)](https://web.dev/articles/cls): 페이지의 전체 생명 주기에서 layout이 변형되는 정도를 측정한 값. nextjs는 font, image 등을 내부적으로 처리하여 CLS를 최적화 해둠
4. [Image component in nextjs](https://nextjs.org/docs/app/building-your-application/optimizing/images)
	- nextjs config 파일에서 default loading 이미지를 지정할 수 있음. 혹은 개별로도 지정 가능
		```jsx
		module.exports = { images: { loader: 'custom', loaderFile: './my/image/loader.js', },}
		```
	- priority를 지정해주면 해당 이미지의 로딩 우선순위가 올라감
		```jsx
		import Image from 'next/image'
		import profilePic from '../public/me.png'
		
		export default function Page() { return <Image src={profilePic} alt="Picture of the author" priority />}
		```
	- width/height를 알 수 없는 경우, fill 속성을 지정해주면 부모의 너비를 따라감
		When using `fill`, the parent element must have `position: relative`
		When using `fill`, the parent element must have `display: block`
		```
		fill={true} // {true} | {false}
		```
	- Image 컴포넌트를 이용한 구현 예시
		- responsive
			```jsx
			import Image from 'next/image'
			import mountains from '../public/mountains.jpg'
			 
			export default function Responsive() {
			  return (
			    <div style={{ display: 'flex', flexDirection: 'column' }}>
			      <Image
			        alt="Mountains"
			        // Importing an image will
			        // automatically set the width and height
			        src={mountains}
			        sizes="100vw"
			        // Make the image display full width
			        style={{
			          width: '100%',
			          height: 'auto',
			        }}
			      />
			    </div>
			  )
			}
			```
		- fill 
			```jsx
			import Image from 'next/image'
			import mountains from '../public/mountains.jpg'
			 
			export default function Fill() {
			  return (
			    <div
			      style={{
			        display: 'grid',
			        gridGap: '8px',
			        gridTemplateColumns: 'repeat(auto-fit, minmax(400px, auto))',
			      }}    >			
			      <div style={{ position: 'relative', height: '400px' }}>
			        <Image
			          alt="Mountains"
			          src={mountains}
			          fill
			          sizes="(min-width: 808px) 50vw, 100vw"
			          style={{
			            objectFit: 'cover', // cover, contain, none
			          }}
			        />
			      </div>
			      {/* And more images in the grid... */}
			    </div>
			  )
			}
			```
		- background-image
			```jsx
				import Image from 'next/image'
				import mountains from '../public/mountains.jpg'
				 
				export default function Background() {
				  return (
				    <Image
				      alt="Mountains"
				      src={mountains}
				      placeholder="blur"
				      quality={100}
				      fill
				      sizes="100vw"
				      style={{
				        objectFit: 'cover',
				      }}
				    />
				  )
				}
				```
	- [Image 컴포넌트를 이용한 구현 예시 (2)(github 포함)](https://image-component.nextjs.gallery/)
		- 특히 color 이게 개쩔음
5. [TS - Partial, Pick, Omit 나만 몰랐겠지](https://kyounghwan01.github.io/blog/TS/fundamentals/utility-types/#%E1%84%8B%E1%85%A8%E1%84%89%E1%85%B5-2)
6. z-index -1 가능한거 처음 알았음
	```css
	.bgWrap {
	
	position: fixed;
	
	height: 100vh;
	
	width: 100vw;
	
	overflow: hidden;
	
	z-index: -1;
	
	}
	```
	
7. [왜 router.push 말고 Link 를 써야할까?](https://stackoverflow.com/questions/74959895/why-using-link-is-better-than-router-push)
# 오늘의 영단어
1. collisions: 충돌 (한 메모리에 두가지 이상의 records가 위치할 경우)
2. populate: ~를 채우다 (fill과는 다름. CS에서는 주로 populate를 사용)
3. fetch: go for and bring back
4. load: carry
5. cumulative: 누적하는
6. implement: put into effort
7. malic: 악의
8. Implicitly: 절대적으로, 은연중에, 암시적으로
9. nested: 보금자리를 마련하다