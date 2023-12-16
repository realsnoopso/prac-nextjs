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

# 보안
https://nextjs.org/blog/security-nextjs-server-components-actions
위 문서 꼭 읽어보자 시간 날 때!

# 오늘의 단어
- collision 충돌 (ID collision)