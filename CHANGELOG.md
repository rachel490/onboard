# @toktokhan-dev/next-page-init

## 0.0.8

### Patch Changes

- c72dced: update s3-file-uploader

  ## @toktokhan-dev 모듈 버젼 update

  프로젝트 내에 사용되고 있는 @toktokhan-dev 모듈들의 버젼이 일괄 업데이트 됐습니다.

  ## axios instance 주석처리

  기존에 제거되있던 refresh flow 의 주석을 다시 추가했습니다.

  ## s3-file-uploader 모듈 변경사항

  똑똑한 개발자의 서버 s3 구현사항이 달라짐에 따라, s3-file-upload 모듈이 수정되었습니다.

  ### Server & Client Flow 상 변경점

  1. backend server 로 부터 받는 presigned-url api 응답 schema 가 달라졌습니다. query param 으로 전달되던 값이 response body 의 fields 로 변경되었습니다.

  #### 기존

  ```json
  {
    "url": "https://s3.ap-northeast-2.amazonaws.com/toktokhan-dev/Key=...&SomeField=..."
  }
  ```

  #### 변경

  ```json
  {
    "url": "https://s3.ap-northeast-2.amazonaws.com/toktokhan-dev/Key=...",
    "fields": {
      "key": "...",
      "SomeField": "..."
    }
  }
  ```

  2. s3 로 upload 시 요청 method 와 reqeuest type 이 변경되었습니다.

  #### 기존

  - method: `PUT`
  - request-type: `File Object`

  ```ts
  const { url , fields } = await presignedUrlApi.createPresignedUrl({...});

  fetch(url, {
    method: 'PUT',
    body: file
  });

  ```

  #### 변경

  - method: `POST`
  - request-type: `FormData`

  변경된 요청 방법은 presigned-url 의 응답으로 받아온 fields 와 File, Content-Type 을 포함하는 `FormData` 로 만들어서 전달합니다.

  ```ts

  const { url , fields } = await presignedUrlApi.createPresignedUrl({...});

  const formData = new FormData();

  Object.entries(fields).forEach(([key, value]) => {
    formData.append(key, value);
  });

  formData.append('file', file);
  formData.append('Content-Type', file.type);

  fetch(url, {
    method: 'POST',
    body: formData
  });

  ```

  ### Client S3FileUploader 모듈 변경점

  #### 기존

  기존에는 presigned url 을 프로젝트 서버로 요청해 생성후에, s3 에 파일을 업로드 하는 플로우를 한번에 처리하기위해 아래와 같은 모듈이 있었습니다.

  ```ts
  class S3FileUploader {
      private _createPresignedUrl () {...}
      private _uploadFileToS3 () {...}

      async uploadFile (file: File) {
          const { url, fields } = await this._createPresignedUrl();
          await this._uploadFileToS3(url, fields, file);
      }

      async uploadFiles (file: File[]) {
          const { url, fields } = await this._createPresignedUrl();
          const result =  Promise.allSattled(file.map(uplaodFileToS3));
          return {
              fulfilled: result.filter(r => r.status === 'fulfilled'),
              rejected: result.filter(r => r.status === 'rejected')
          }
      }
  }
  ```

  #### 변경

  위의 방식은 gen:api 를 통해 생성된 모듈을 사용하지 못한다는 단점이 있고, File 객체의 형식이 다른 react-native 를 포함해 프로젝트마다 달라 질 수 있는 구현 사항에 따라,
  S3FileUploaderApi 모듈을 직접 수정하게 될 여지가 있다는 단점이 있었습니다.

  따라서 아래처럼, 사용되는 함수를 주입받아 flow 만 처리하는 모듈이 새롭게 만들어 졌고, S3FileUplader 모듈은 오직 s3 에 파일을 업로드 하는 역할만을 수행하도록 변경되었습니다.

  ```ts
  // S3FileUpladerApi.ts
  class S3FileUploaderApi {
      async uploadFileToS3 ({ url, formData }) {...}
  }
  // S3FileUploaderApi.query.ts
  export const { uploadFile, uploadFiles } = createS3UploadFlow({
    // 아래 부분은 프로젝트 상황에 맞게 직접 작성합니다.
    prepareUpload: async (file: File) => {
      const { name, type } = file
      // gen:api 로 생성된  presigned url 을 생성하는 api 를 직접 import 해서 사용합니다.
      const { fields, url } = await presignedUrlApi.presignedUrlCreate({ name, type })
      const formData = new FormData()

      Object.entries(fields).forEach(([k, v]) => formData.append(k, v))
      formData.append('Content-Type', file.type)
      formData.append('file', file)

      return {
        url,
        formData,
        fields,
        file,
      }
    },
    // 아래 부분은 프로젝트 상황에 맞게 직접 작성합니다.
    uploadFileToS3: async ({ url, formData, file, fields }) => {
      await s3FileUploaderApi.uploadFileToS3({ url, formData })
      ...
    },
  })

  const useS3FileUploadMutate = (...) => useMutation({ mutationFn: uploadFile })
  ```

## 0.0.7

### Patch Changes

- 4f18ba6: isLogin 이 null 일때를 고려하여 ClientOnly 컴포넌트를 생성해 관리하는 로직이 추가되었습니다.
  아래 같은 경우는 isLogin 이 null 일때, falsy 한 값으로 간주하여, 로그인 버튼을 랜더링 하게 됩니다.

  ```tsx
  {
    isLogin ?
      <Button variant={'line'} size={'sm'} onClick={() => {}}>
        Logout
      </Button>
    : <Link variant={'line'} size={'sm'} href={ROUTES.LOGIN_MAIN}>
        Login
      </Link>
  }
  ```

  현재 토큰은 localStorage 에 저장되어 있지만, 서버에서 localStorage 는 접근 할수 없기 때문에, 하이드레이션이 일어나기전 서버에서 최로로 받은 코드는 토큰이 존재하지 않은 상태가 되게 됩니다.
  따라서, isLogin 값은 서버에선 의도적으로 null 을 할당 받게 되어있습니다.

  ```tsx
  const isLogin: boolean | null = isClient ? !!token?.access_token : null
  ```

  로그인 뿐 아니라 로컬 스토리지 관련 값을 사용할때는 `Null` 상태를 고려하여 최초 하이드레이션이 일어나기 전 시점에는 아예 랜더링을 안하는 쪽이 좋습니다.

  ```tsx
  () => ({
  if(isLogin === null) return null
  return isLogin ? <Button>Logout</Button> : <Button><Login/Button>
  })()
  ```

  똑똑한 개발자의 보일러플레이트에는 선언적으로 처리하기 위해 useClient 훅을 사용하여 `ClientOnly` 컴포넌트를 만들어 핸들링하는 로직이 추가되었습니다.

  ```tsx
  interface ClientOnlyProps {
    children: React.ReactNode
    fallback?: React.ReactNode
  }

  const ClientOnly = ({ children, fallback }: ClientOnlyProps) => {
    const isClient = useClient()
    if (!isClient) return isNullish(fallback) ? null : fallback

    return children
  }
  ```

  사용

  ```tsx
  <ClientOnly>
  { isLogin ? <Button>Logout</Button> : <Button>Login</Button }
  </ClientOnly>
  ```

## 0.0.6

### Patch Changes

- 57bdbfc: improve retry-request-manager

  ### clenupTimeout option 추가

  동시적인 expired 된 요청이 많을 때 refresh 요청이 2번 호출되는 이슈가 있었습니다.

  이유는 리프레시 갱신전 요청들 중, 이미 이전에 도착한 만료응답에 의해 [토큰 갱신 - 재요청] 플로우가 완료 된 후, 응답된 요청이 있다면 refresh flow 를 다시 타는것이 원인 이었습니다.

  예를들어

  - A 요청 - 만료 응답까지 1초
    - A 요청 refresh - 1초
    - A 요청 재요청 - 1초
    - refesh flow 초기화
  - B 요청 - 만료 응답까지 5초

  의 경우 A 요청 같은경우는 재요청 까지 총 3초가 걸리기 때문에 B 요청 응답 전에 리프레시 플로우가 완료(초기화)가됩니다.

  B 요청의 응답이 도착한 시점에는 이미 리프레시 플로우가 초기화 된 시점이기 때문에 리프레시가 2번 발생할 수 있습니다.

  따라서, 토큰을 정리하기 까지 timeout 으로 시간 여유를 주어 이슈 발생 확률을 낮추도록 개선 했습니다.

  요청이 여러개일 경우 정리 함수의 timeout 은 이전 것을 정리하고, 새롭게 timeout 을 설정하기 때문에, debounce 처럼 가장 마지막에 설정된 timout 함수만 지연된 시간으로 실행하게 됩니다.

  ```ts
  // configs/axios/instance.ts
  const retry = retryRequestManager({ cleanupTimeOut: 5000 })
  ```

  ```ts
  // configs/axios/retry-request-manager.ts
  export const retryRequestManager = (options?: {
    /**
     * 토큰 refresh 후, 설정한 시간안에 refresh 를 다시 시도 하지 않을경우 토큰을 삭제합니다.
     * @default 0
     *  */
    cleanupTimeOut?: number
  }) => {
    const { cleanupTimeOut = 0 } = options || {}
    let token: Promise<string> | null = null
    let timeoutId: NodeJS.Timeout | null = null

    return async (params: {
      getToken: () => Promise<string>
      onRefetch: (refreshed: string) => any
      onError: (error: any) => any
    }) => {
      try {
        ...
      } catch (err) {
        ...
      } finally {
        if (timeoutId) clearTimeout(timeoutId)
        if (token) {
          timeoutId = setTimeout(() => {
            token = null
          }, cleanupTimeOut)
        }
      }
    }
  }
  ```

## 0.0.5

### Patch Changes

- 73246c3: axios refresh & Token Naming

  # Axios Instance, Refresh

  기존 retryRequestManager 의 함수명에서 오타 수정 및 파일 이름이 변경되었습니다.

  # Token Naming

  기존 localStorage 의 token 키에 저장되던 객체의 property naming 이 수정되었습니다.

  #### 기존

  ```ts
  {
    access: string
    refresh: string
  }
  ```

  #### 변경

  ```ts
  {
    access_token: string
    refresh_token: string
  }
  ```

## 0.0.4

### Patch Changes

- 2eb1b20: update react-web

  @toktokhan-dev/react-web package 가 0.0.8 에서 0.0.11 로 수정 되었습니다.

  social 로그인 관련 api 들이 수정됨에 따라 예시가 수정되었습니다.

## 0.0.3

### Patch Changes

- 5fedf7f: - update'@toktokhan-dev/\*' packages
  - update social oauth
    - onFail parameters
  - update example react query v5대응
  - eslint downgrade v8.5.3

## 0.0.2

### Patch Changes

- 7e3e050: add gitignore
- 7ea6357: remove pnpm-lock.yaml
- b8b2bf3: add tok-cli-commit
- 2bf5a83: update packages version

## 0.0.1

### Patch Changes

- 9ff916e: 똑똑한 개발자 컨벤션이 담긴 Next.js page-router 가 적용 된 보일러 플레이트 입니다.

## 0.0.3

### Patch Changes

- a96412c: changeset action for git tag
- 05eafad: changeset action version -> publish

## 0.0.2

### Patch Changes

- 621ee4f: changeset action for git tag

## 0.0.1

### Patch Changes

- 58f516e: use pnpm in github action
- c21709a: changeset pakcage name
- f11c21e: package.json name
- 270d669: setup husky
- 3194fde: initialize
