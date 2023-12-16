# 새로 알게 된 내용

- 실제로 재렌더가 안일어남.
	- 아마 await를 내부에서 사용하지 않아서 일지도..?
	- 그래서 dynamic하게 바꿔주려고 억지로 param을 쓰게 됨 (updateAt으로 현재 시간을 강제로 집어넣어서 변경이 일어나게 만듦)
	- dynamic 하게 다루려면 param을 다르게 넣는 방법밖에 없을까?
		- 12 챕터 열심히 안 읽은 대가를 치룸 ㅋㅋㅋㅋ
- https://nextjsconf-pics.vercel.app/
	- 주소를 가짜로 넣고, 새로고침 했을 땐 실제 주소로 이동하는 방법으로 설계됨
	- https://github.com/vercel/next.js/tree/canary/examples/with-cloudinary
	- Link 말고 button에서 할 수 있는 방법 없을까..?
# Mutating Data
- React Server Actions는 서버에서 다이렉트로 작동하는 비동기 코드
- 서버 액션이 생겨난 이유는 Security 때문임
	- POST requests, encrypted clousers, strict input checks, error message hasing, host restrictions 등을 지원하고자 함
- next 에서 <form/> 을 사용할 때 action 이라는 것을 사용하면 formdata object를 받을 수 있게 됨
- 이렇게 되면 js 가 작동하지 않는 환경에서도 작동시킬 수 있는 장점이 있음
- 또, caching 과도 밀접하게 연계되는 장점이 있음
	- revalidatePath, revalidateTag 등의 API를 사용
```js
'use client';
 
import { customerField } from '@/app/lib/definitions';
import Link from 'next/link';
import {
  CheckIcon,
  ClockIcon,
  CurrencyDollarIcon,
  UserCircleIcon,
} from '@heroicons/react/24/outline';
import { Button } from '@/app/ui/button';
import { createInvoice } from '@/app/lib/actions';
 
export default function Form({
  customers,
}: {
  customers: customerField[];
}) {
  return (
    <form action={createInvoice}>
      // ...
  )
}
```

HTML에서는 action에 URL을 넘김. 하지만 React에서는 action attribute가 특별한 prop이 됨. Server Actions는 POST API endpoint를 생성함. 그래서 따로 API endpoint를 생성할 필요가 없는거

![[Screenshot 2023-12-16 at 5.56.29 PM.png]]
![[Screenshot 2023-12-16 at 5.56.36 PM.png]]
post 요청이 해당 페이지로 날아감. 그런데 form의 action attribute에서는 허용되고 다른 곳에서는 안된다. 더 자세한 작동 원리까진 모르게씀..

이제 formData를 추출해보자!

들어온 formData를 그냥 추출하기만 하면됨.
기본 JS API 그냥 사용하기

```js
'use server';
 
export async function createInvoice(formData: FormData) {
  const rawFormData = {
    customerId: formData.get('customerId'),
    amount: formData.get('amount'),
    status: formData.get('status'),
  };
  // Test it out:
  console.log(rawFormData);
}
```

Object.formEntries() 를 사용해도 됨
 `const rawFormData = Object.fromEntries(formData.entries())`
https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/fromEntries

이제 validation을 할 수 있음

```js
export type Invoice = {
  id: string; // Will be created on the database
  customer_id: string;
  amount: number; // Stored in cents
  status: 'pending' | 'paid';
  date: string;
};
```


이렇게 정의한 스키마가 검증을 도와준다
coerce method는 string을 number로 변환시켜줌

```js
'use server';
 
import { z } from 'zod';
import { sql } from '@vercel/postgres';
import { revalidatePath } from 'next/cache';
 
const FormSchema = z.object({
  id: z.string(),
  customerId: z.string(),
  amount: z.coerce.number(),
  status: z.enum(['pending', 'paid']),
  date: z.string(),
});
 
const CreateInvoice = FormSchema.omit({ id: true, date: true });


export async function createInvoice(formData: FormData) {
	// 검증이 끝난 다음 넘겨주기
	const { customerId, amount, status } = CreateInvoice.parse({ 
		customerId: formData.get('customerId'), 
		amount: formData.get('amount'), 
		status: formData.get('status'), });
		
		const amountInCents = amount * 100; 
		const date = new Date().toISOString().split('T')[0];

		await sql` INSERT INTO invoices (customer_id, amount, status, date) 
		VALUES (${customerId}, ${amountInCents}, ${status}, ${date}) 
		
		revalidatePath('/dashboard/invoices');
		redirect('/dashboard/invoices');
		`;
	}
```

- 좀 더 안전하게 짜기 위해서 단위를 cents로 설정해주면 부동소수점 오류를 없애고 정확도를 높일 수 있음
- Date 형식 변환해주기

- 가장 중요한 Revalidate!!
	- Next는 Client-side Router Cache를 갖고 있어서 라우트 세그먼트를 브라우저에 일정 시간동안 저장하게 된다. prefetching과 더불어 이 캐시는 user가 빠르게 탐색할 수 있게 도와줌. 
	- invoices route를 업데이트하고 있다면, cache를 지우고 싶을 것. 그러면 'revalidatePath' API를 사용해야 함!
	- 그럼 캐시가 날아가고 새로운 데이터가 서버에서 가져와짐
	- 이 다음 redirect를 할 것.

이제 다시 updating으로.

[id] 와 같은 폴더명으로 dynamic routing 구현 가능


edit 페이지는 이렇게. 여러 데이터 한꺼번에 받아오기 위해 Promise.all 때려버리기.

```js
import Form from '@/app/ui/invoices/edit-form';
import Breadcrumbs from '@/app/ui/invoices/breadcrumbs';
import { fetchCustomers } from '@/app/lib/data';
 
export default async function Page({ params }: { params: { id: string } }) { 
	const id = params.id;
	const [invoice, customers] = await Promise.all([
		fetchInvoiceById(id),	
		fetchCustomers(),
	]);
  return (
    <main>
      <Breadcrumbs
        breadcrumbs={[
          { label: 'Invoices', href: '/dashboard/invoices' },
          {
            label: 'Edit Invoice',
            href: `/dashboard/invoices/${id}/edit`,
            active: true,
          },
        ]}
      />
      <Form invoice={invoice} customers={customers} />
    </main>
  );
}
```

create 페이지는 이렇게. 컴포넌트 만들고 외부에서 데이터를 주입해주는 방식이 훨씬 깔끔하고 정석적인듯.

```js
import Form from '@/app/ui/invoices/create-form';
import Breadcrumbs from '@/app/ui/invoices/breadcrumbs';
import { fetchCustomers } from '@/app/lib/data';

export default async function Page() {
	const customers = await fetchCustomers();
	return (
	<main>
		<Breadcrumbs breadcrumbs={[
			{ label: 'Invoices', href: '/dashboard/invoices' },
			{
				label: 'Create Invoice',
				href: '/dashboard/invoices/create',
				active: true,
			},
		]}/>
		<Form customers={customers} />
	</main>
	);
}
```

UUIDs vs Auto-incrementing Keys
- UUIDs 를 사용하는 이유는 ID 충돌의 위험을 줄여주기 때문. 그리고 열거 공격을 당할 수도 있음.
- 근데 URL 예쁘게 만들려면 auto-incrementing 써도 되긴 함

마찬가지로 form action 프로퍼티에 function 주입하기

```js
// Use Zod to update the expected types
const UpdateInvoice = FormSchema.omit({ id: true, date: true });
 
// ...
 
export async function updateInvoice(id: string, formData: FormData) {
  const { customerId, amount, status } = UpdateInvoice.parse({
    customerId: formData.get('customerId'),
    amount: formData.get('amount'),
    status: formData.get('status'),
  });
 
  const amountInCents = amount * 100;
 
  await sql`
    UPDATE invoices
    SET customer_id = ${customerId}, amount = ${amountInCents}, status = ${status}
    WHERE id = ${id}
  `;
 
  revalidatePath('/dashboard/invoices');
  redirect('/dashboard/invoices');
}
```

### 보안
https://nextjs.org/blog/security-nextjs-server-components-actions
위 문서 꼭 읽어보자 시간 날 때!

# Handling Errors
- 엥 왜 try 안에 안넣지..?
```js
export async function createInvoice(formData: FormData) {
  const { customerId, amount, status } = CreateInvoice.parse({
    customerId: formData.get('customerId'),
    amount: formData.get('amount'),
    status: formData.get('status'),
  });
 
  const amountInCents = amount * 100;
  const date = new Date().toISOString().split('T')[0];
 
  try {
    await sql`
      INSERT INTO invoices (customer_id, amount, status, date)
      VALUES (${customerId}, ${amountInCents}, ${status}, ${date})
    `;
  } catch (error) {
    return {
      message: 'Database Error: Failed to Create Invoice.',
    };
  }
 
  revalidatePath('/dashboard/invoices');
  redirect('/dashboard/invoices');
}
```

- redirect가 try/catch 블록 외부에 있는 이유는 redirect가 throwing error를 던질 수 있어서임 (이 친구도 promise였던 것 같음) redirect를 그래서 try/catch 문 바깥에 두는 것.

그런데 try catch 문 바깥에서 갑작스럽게 나타낸 에러가 있을 수 있음.
그래서 error.tsx를 사용함

```js
'use client';
 
import { useEffect } from 'react';
 
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Optionally log the error to an error reporting service
    console.error(error);
  }, [error]);
 
  return (
    <main className="flex h-full flex-col items-center justify-center">
      <h2 className="text-center">Something went wrong!</h2>
      <button
        className="mt-4 rounded-md bg-blue-500 px-4 py-2 text-sm text-white transition-colors hover:bg-blue-400"
        onClick={
          // Attempt to recover by trying to re-render the invoices route
          () => reset()
        }
      >
        Try again
      </button>
    </main>
  );
}
```

- error : 에러 객체를 받음
- rest: error boundary를 reset할 수 있는 함수. 해당 route segment를 재렌더하는 것을 시도하게 됨

- 없는 경로로 이동하게 되면 error.tsx가 에러를 잡게 됨. notFound 메소드를 적절히 써야 함
```js
import { fetchInvoiceById, fetchCustomers } from '@/app/lib/data';
import { updateInvoice } from '@/app/lib/actions';
import { notFound } from 'next/navigation';
 
export default async function Page({ params }: { params: { id: string } }) {
  const id = params.id;
  const [invoice, customers] = await Promise.all([
    fetchInvoiceById(id),
    fetchCustomers(),
  ]);
 
  if (!invoice) {
    notFound();
  }
 
  // ...
}
```

이렇게 지정해주고, not-found.tsx 파일을 폴더 내부에 만들어주면 그 컴포넌트가 렌더링되게 됨.

```js
import Link from 'next/link';
import { FaceFrownIcon } from '@heroicons/react/24/outline';
 
export default function NotFound() {
  return (
    <main className="flex h-full flex-col items-center justify-center gap-2">
      <FaceFrownIcon className="w-10 text-gray-400" />
      <h2 className="text-xl font-semibold">404 Not Found</h2>
      <p>Could not find the requested invoice.</p>
      <Link
        href="/dashboard/invoices"
        className="mt-4 rounded-md bg-blue-500 px-4 py-2 text-sm text-white transition-colors hover:bg-blue-400"
      >
        Go Back
      </Link>
    </main>
  );
}
```

# Improving Accessibility
https://web.dev/learn/accessibility/
이거 읽어보자!!

- default로 Next.js가 eslint-plugin-jsx-ally 플러그인을 사용해서 accessibility 이슈를 잡아줌.
- alt 텍스트가 없으면 `aria-*`, role을 사용하라고 경고해줌
- next lint 실행하면 됨

그럼 form accessibility 높이려면 어떻게 해야할까


AT support (assistive technologies)
- semantic HTML: div 쓰지 말기. input, option, etc 이런 태그 쓰기. 왜 div 쓰면 안되는지 알았다... (자동으로 너비 조절되는 input 만들때 div가 더 만들기 편해서 쓰려고 했는데...) 
- labelling: <label/> 과 htmlFor 속성은 form field의 속성을 명시적으로 보여줌. 
- focus outline: 포커싱 될 때 구분 잘 해주기. tab으로 가능한지 확인할 수 있음

form validation
null을 제출하면 에러가 남
그래서 서버 제출 전에 항상 검증을 해야 함

require attribute를 추가해서 검증을 강제 할 수 있음 (브라우저에서 제공하는 기능)

비어있으면 자동으로 알람
이게 클라이언트 사이드에서 제어하는 법.

<input/> 및 <select/>에서 사용 가능

```js
<input
  id="amount"
  name="amount"
  type="number"
  placeholder="Enter USD amount"
  className="peer block w-full rounded-md border border-gray-200 py-2 pl-10 text-sm outline-2 placeholder:text-gray-500"
  required
/>
```
![[Screenshot 2023-12-16 at 3.34.52 PM.png]]
server side validation은 어떻게 할까?

이제부터 useFormState를 사용할 것임. (react-dom의)

```js
export async function createInvoice(prevState: State, formData: FormData) {
  // Validate form using Zod
  const validatedFields = CreateInvoice.safeParse({
    customerId: formData.get('customerId'),
    amount: formData.get('amount'),
    status: formData.get('status'),
  });
 
  // If form validation fails, return errors early. Otherwise, continue.
  if (!validatedFields.success) {
    return {
      errors: validatedFields.error.flatten().fieldErrors,
      message: 'Missing Fields. Failed to Create Invoice.',
    };
  }
 
  // Prepare data for insertion into the database
  const { customerId, amount, status } = validatedFields.data;
  const amountInCents = amount * 100;
  const date = new Date().toISOString().split('T')[0];
 
  // Insert data into the database
  try {
    await sql`
      INSERT INTO invoices (customer_id, amount, status, date)
      VALUES (${customerId}, ${amountInCents}, ${status}, ${date})
    `;
  } catch (error) {
    // If a database error occurs, return a more specific error.
    return {
      message: 'Database Error: Failed to Create Invoice.',
    };
  }
 
  // Revalidate the cache for the invoices page and redirect the user.
  revalidatePath('/dashboard/invoices');
  redirect('/dashboard/invoices');
}
```

이렇게 하면 깔끔하게 formData를 검증할 수 있음

```js
<form action={dispatch}>
  <div className="rounded-md bg-gray-50 p-4 md:p-6">
    {/* Customer Name */}
    <div className="mb-4">
      <label htmlFor="customer" className="mb-2 block text-sm font-medium">
        Choose customer
      </label>
      <div className="relative">
        <select
          id="customer"
          name="customerId"
          className="peer block w-full rounded-md border border-gray-200 py-2 pl-10 text-sm outline-2 placeholder:text-gray-500"
          defaultValue=""
          aria-describedby="customer-error"
        >
          <option value="" disabled>
            Select a customer
          </option>
          {customerNames.map((name) => (
            <option key={name.id} value={name.id}>
              {name.name}
            </option>
          ))}
        </select>
        <UserCircleIcon className="pointer-events-none absolute left-3 top-1/2 h-[18px] w-[18px] -translate-y-1/2 text-gray-500" />
      </div>
      <div id="customer-error" aria-live="polite" aria-atomic="true">
        {state.errors?.customerId &&
          state.errors.customerId.map((error: string) => (
            <p className="mt-2 text-sm text-red-500" key={error}>
              {error}
            </p>
          ))}
      </div>
    </div>
    // ...
  </div>
</form>
```

- `aria-describedby="customer-error"`: select 엘리먼트와 error message container 사이의 관계를 만들어줌.  id가 customer-error 인 것끼리 매칭이 됨. 
- `id="customer-error"`: 이게 식별자
- `aria-live="polite"`: 에러가 있음을 스크린 리더가 공손하게 알려줌(?) 방해 받지 않도록 알리는 것이 포인트인 듯. 

  
  

# 15. Adding Authentication

- 검증이란 시스템이 유저가 유저인지 검증하는 것을 말함
- username과 패스워드 확인, 혹은 verification code 보내기.. 2FA 는 보안을 증가시켜줌


Authentication: 유저가 유저인지 확인하는 것. 신원을 확인. username과 password를 건넴으로써. (누구인지)
Authorization: 아이덴티티가 확인되면, 어플리케이션의 어떠한 범위까지 허락할 것인지를 결정하는 것 (뭘 할 수 있는가)

NextAuth 라이브러리 사용하라고 추천함

```js
npm install next-auth@beta
```

14 버전이랑 호환되는 건 아직 beta 버전이라고는 함.

```
openssl rand -base64 32
```

터미널에서 위 명령어 실행해서 secret key 만들기
```
AUTH_SECRET=your-secret-key
```

vercel 환경 변수에도 넣어주기!
그 다음은 page option 만들기

```js
import type { NextAuthConfig } from 'next-auth';
	export const authConfig = {
	pages: {
		signIn: '/login',
	},
};

```

  

이 설정으로 custom sign-in, sign-out, error 페이지를 설정 가능함

이거 설정하지 않으면 NextAuth.js가 설정한 default page로 떨어지게 됨

  

다음 Next.js Midddleware 로 routes 보호하기!

  

```js
import type { NextAuthConfig } from 'next-auth';
 
export const authConfig = {
  pages: {
    signIn: '/login',
  },
  callbacks: {
    authorized({ auth, request: { nextUrl } }) {
      const isLoggedIn = !!auth?.user;
      const isOnDashboard = nextUrl.pathname.startsWith('/dashboard');
      if (isOnDashboard) {
        if (isLoggedIn) return true;
        return false; // Redirect unauthenticated users to login page
      } else if (isLoggedIn) {
        return Response.redirect(new URL('/dashboard', nextUrl));
      }
      return true;
    },
  },
  providers: [], // Add providers with an empty array for now
} satisfies NextAuthConfig;
```

  

routes를 보호해주는 역할을 함. 로그인하지 않은 유저가 dashboard로 접근하는 것을 막아줌

- authorized 콜백은 request가 authorized 되어 있는지 검증함. (Middleware를 통해서!)

- auth, request properties를 받음

- auth property는 user 세션을 포함함. request property는 imcoming request를 포함

- providers 옵션은 array임. login option을 설정 가능.

- https://nextjs.org/learn/dashboard-app/adding-authentication#adding-the-credentials-provider

- 위 링크 참고

- 참고
	
	```js
	
	import NextAuth from "next-auth"
	
	import CredentialsProvider from "next-auth/providers/credentials"
	
	export default NextAuth({
	
	providers: [
	
	CredentialsProvider({
	
	async authorize(credentials) {
	
	const authResponse = await fetch("/users/login", {
	
	method: "POST",
	
	headers: {
	
	"Content-Type": "application/json",
	
	},
	
	body: JSON.stringify(credentials),
	
	})
	
	if (!authResponse.ok) {
	
	return null
	
	}
	
	const user = await authResponse.json()
	
	return user
	
	},
	
	}),
	
	],
	
	})
	
	...
	
	import NextAuth from 'next-auth';
	
	import { authConfig } from './auth.config';
	
	import Credentials from 'next-auth/providers/credentials';
	
	export const { auth, signIn, signOut } = NextAuth({
	
	...authConfig,
	
	providers: [Credentials({})],
	
	});
	
	```
	  

```js
import type { NextAuthConfig } from 'next-auth';
 
export const authConfig = {
  pages: {
    signIn: '/login',
  },
  callbacks: {
    authorized({ auth, request: { nextUrl } }) {
      const isLoggedIn = !!auth?.user;
      const isOnDashboard = nextUrl.pathname.startsWith('/dashboard');
      if (isOnDashboard) {
        if (isLoggedIn) return true;
        return false; // Redirect unauthenticated users to login page
      } else if (isLoggedIn) {
        return Response.redirect(new URL('/dashboard', nextUrl));
      }
      return true;
    },
  },
  providers: [], // Add providers with an empty array for now
} satisfies NextAuthConfig;
```

  

```js

// middleware.ts

import NextAuth from "next-auth";
import { authConfig } from "./auth.config";
export default NextAuth(authConfig).auth;

export const config = {
	// https://nextjs.org/docs/app/building-your-application/routing/middleware#matcher
	matcher: ["/((?!api|_next/static|_next/image|.*\\.png$).*)"],
};
```

- 미들웨어에서 matcher로 특정한 path 에서만 실행 가능하게 변경 가능함

- 여기에 적힌 path는 접속하기 전에 무조건 미들웨어로 먼저 들렸다감

  

그다음 auth.ts 파일 생성해서 authConfig를 넣어줌.

  

Credentials도 설정해줌

- 유저가 username과 password로 로그인하게 도와줌

- 꼭 Credentials 를 써야 하는건 아니고 Oauth, email도 좋은 대안이 될 수 있음

- Although we're using the Credentials provider, it's generally recommended to use alternative providers such as [OAuth](https://authjs.dev/getting-started/providers/oauth-tutorial) or [email](https://authjs.dev/getting-started/providers/email-tutorial) providers. See the [NextAuth.js docs](https://authjs.dev/getting-started/providers) for a full list of options.

  

Validation도 잘 해주기!

```js
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import { authConfig } from './auth.config';
import { sql } from '@vercel/postgres';
import { z } from 'zod';
import type { User } from '@/app/lib/definitions';
import bcrypt from 'bcrypt';
import { sql } from '@vercel/postgres';
import type { User } from '@/app/lib/definitions';
import bcrypt from 'bcrypt';

async function getUser(email: string): Promise<User | undefined> { 
	try { const user = await sql<User>`SELECT * FROM users WHERE email=${email}`; return user.rows[0];` } 
	catch (error) { console.error('Failed to fetch user:', error); throw new Error('Failed to fetch user.'); 
}}
 
// ...
 
export const { auth, signIn, signOut } = NextAuth({
  ...authConfig,
  providers: [
    Credentials({
      async authorize(credentials) {
        if (parsedCredentials.success) { 
	        const { email, password } = parsedCredentials.data; 
	        const user = await getUser(email); 
	        if (!user) return null; 
		}
 
        if (parsedCredentials.success) {
          const { email, password } = parsedCredentials.data;
          const user = await getUser(email);
          if (!user) return null;
          const passwordsMatch = await bcrypt.compare(password, user.password);
 
          if (passwordsMatch) return user;
        }
 
        console.log('Invalid credentials');
        return null;
      },
    }),
  ],
});
```

bcrypt.compare 를 사용하여 password 체크


```js
import { signIn } from '@/auth';
import { AuthError } from 'next-auth';
 
// ...
 
export async function authenticate(
  prevState: string | undefined,
  formData: FormData,
) {
  try {
    await signIn('credentials', formData);
  } catch (error) {
    if (error instanceof AuthError) {
      switch (error.type) {
        case 'CredentialsSignin':
          return 'Invalid credentials.';
        default:
          return 'Something went wrong.';
      }
    }
    throw error;
  }
}
```


그 다음 action 생성해주기
```js
import { signIn } from '@/auth';
import { AuthError } from 'next-auth';
 
// ...
 
export async function authenticate(
  prevState: string | undefined,
  formData: FormData,
) {
  try {
    await signIn('credentials', formData);
  } catch (error) {
    if (error instanceof AuthError) {
      switch (error.type) {
        case 'CredentialsSignin':
          return 'Invalid credentials.';
        default:
          return 'Something went wrong.';
      }
    }
    throw error;
  }
}
```


```jsx
'use client';
 
import { lusitana } from '@/app/ui/fonts';
import {
  AtSymbolIcon,
  KeyIcon,
  ExclamationCircleIcon,
} from '@heroicons/react/24/outline';
import { ArrowRightIcon } from '@heroicons/react/20/solid';
import { Button } from '@/app/ui/button';
import { useFormState, useFormStatus } from 'react-dom';
import { authenticate } from '@/app/lib/actions';
 
export default function LoginForm() {
  const [errorMessage, dispatch] = useFormState(authenticate, undefined);
 
  return (
    <form action={dispatch} className="space-y-3">
      <div className="flex-1 rounded-lg bg-gray-50 px-6 pb-4 pt-8">
        <h1 className={`${lusitana.className} mb-3 text-2xl`}>
          Please log in to continue.
        </h1>
        <div className="w-full">
          <div>
            <label
              className="mb-3 mt-5 block text-xs font-medium text-gray-900"
              htmlFor="email"
            >
              Email
            </label>
            <div className="relative">
              <input
                className="peer block w-full rounded-md border border-gray-200 py-[9px] pl-10 text-sm outline-2 placeholder:text-gray-500"
                id="email"
                type="email"
                name="email"
                placeholder="Enter your email address"
                required
              />
              <AtSymbolIcon className="pointer-events-none absolute left-3 top-1/2 h-[18px] w-[18px] -translate-y-1/2 text-gray-500 peer-focus:text-gray-900" />
            </div>
          </div>
          <div className="mt-4">
            <label
              className="mb-3 mt-5 block text-xs font-medium text-gray-900"
              htmlFor="password"
            >
              Password
            </label>
            <div className="relative">
              <input
                className="peer block w-full rounded-md border border-gray-200 py-[9px] pl-10 text-sm outline-2 placeholder:text-gray-500"
                id="password"
                type="password"
                name="password"
                placeholder="Enter password"
                required
                minLength={6}
              />
              <KeyIcon className="pointer-events-none absolute left-3 top-1/2 h-[18px] w-[18px] -translate-y-1/2 text-gray-500 peer-focus:text-gray-900" />
            </div>
          </div>
        </div>
        <LoginButton />
        <div
          className="flex h-8 items-end space-x-1"
          aria-live="polite"
          aria-atomic="true"
        >
          {errorMessage && (
            <>
              <ExclamationCircleIcon className="h-5 w-5 text-red-500" />
              <p className="text-sm text-red-500">{errorMessage}</p>
            </>
          )}
        </div>
      </div>
    </form>
  );
}
 
function LoginButton() {
  const { pending } = useFormStatus();
 
  return (
    <Button className="mt-4 w-full" aria-disabled={pending}>
      Log in <ArrowRightIcon className="ml-auto h-5 w-5 text-gray-50" />
    </Button>
  );
}
```

# 16. Adding Metadata

web development에서는 metadata는 웹페이지에 대한 디테일을 제공해준다. user에게는 안보이지만 HTML 안에 삽입되어 뒤에서 작동함. search engine에게는 상당히 중요함.

SEO에 연관되기 때문에 매우 중요하다. Open graph와 같은 metadata는 shared links 의 형태를 멋지게 만들어줌. 

- title: 웹사이트 타이틀. SEO에 진짜 중요
- description: 사이트의 overview를 제공함
- keyword: 검색 엔진이 page들을 indexing 하는 것에 도움을 줌
	```
	<meta name="keywords" content="keyword1, keyword2, keyword3" />
	```
- open graph metadata: 이 메타데이터는 webpage가 소셜미디어에 공유되었을 때의 형태를 결정함
	```
	<meta property="og:title" content="Title Here" />
	<meta property="og:description" content="Description Here" />
	<meta property="og:image" content="image_url_here" />
	```
- favicon: 파비콘을 설정할 때 사용
	```
	<link rel="icon" href="path/to/favicon.ico" />
	```

두가지 방법으로 구현 가능
- config-based : metadata object export 혹은 generateMetadata function 사용 (layout.js 혹은 page.js)
- file-based
	- `favicon.ico`, `apple-icon.jpg`, and `icon.jpg`: Utilized for favicons and icons
	- `opengraph-image.jpg` and `twitter-image.jpg`: Employed for social media images
	- `robots.txt`: Provides instructions for search engine crawling
	- `sitemap.xml`: Offers information about the website's structure

favicon이랑 open-graph 이미지는 /app 폴더에 넣으면 자동으로 인식됨 (위 형식 지켜서)

dynamic OG 이미지를 생성하려면 다음 문서 참고하기
https://nextjs.org/docs/app/api-reference/functions/image-response


```jsx
// /app/layout.tsx
import { Metadata } from 'next';
 
export const metadata: Metadata = {
  title: {
    template: '%s | Acme Dashboard',
    default: 'Acme Dashboard',
  },
  description: 'The official Next.js Learn Dashboard built with App Router.',
  metadataBase: new URL('https://next-learn-dashboard.vercel.sh'),
};
```

타이틀 템플릿 이용해서 동적으로 내용을 변경 가능함

```jsx
import { Metadata } from 'next';

export const metadata: Metadata = {
	title: 'Invoices',
};
```

이렇게 하면 실제 출력은 Invoices | Acme Dashboard 이렇게 나옴
# 오늘의 단어
- collision 충돌 (ID collision)
- abrupt 불시의, 갑작스러운 (abrupt failure)