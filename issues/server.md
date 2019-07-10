## schema/task/Query.ts, schema/task/Mutation.ts 에서 Type Error
- 아래 파일들 코드 복붙 후 Type Error 나옴
  - [/src/schema/task/Query.ts](https://github.com/geoseong/serverless-graphql-workshop/blob/master/documents/1-graphql/README.md#srcschemataskqueryts), [/src/schema/task/Mutation.ts](https://github.com/geoseong/serverless-graphql-workshop/blob/master/documents/1-graphql/README.md#srcschemataskmutationts)
- `yarn dev`가 켜 있는 상태에서 코드를 복붙하니까 나온 에러
- 시간이 지나서 다시 Save를 하면 에러가 사라짐 (`No type errors found`)
- 에러가 뜨자마자 `yarn dev` 서버를 껐다가 다시 켜면 에러가 사라질 것으로 예상됨.
- 에러코드 위치
  ```ts
  export const TaskMutations = extendType({
    type: 'Mutation',
    definition(t) {
      t.field('createTask', {
        type: 'Task', // <--[error] TS2322: Type '"Task"' is not assignable to type '"Query" | "Mutation" | "Boolean" | "Float" | "ID" | "Int" | "String" | NexusObjectTypeDef<string> | NexusInterfaceTypeDef<string> | NexusUnionTypeDef<string> | NexusEnumTypeDef<string> | NexusScalarTypeDef<...> | NexusWrappedType<...>'.
        args: {
          content: stringArg({
            required: true,
          }),
        },
  ```

## schema/task/Mutation.ts updateTask & deleteTask
### 에러
- playground에서 `deleteTask`를 하면 에러가 나옴.
  - `mutation`
    ```
    mutation del {
      deleteTask(id: "gx54Ei6ZC7wDh7JAwV8uLV") {
        id
        content
      }
    }
    ```
  - `error`
    ```
    {
      "message": "gx54Ei6ZC7wDh7JAwV8uLV라는 ID를 가진 Task를 찾을 수 없습니다",
    }
    ```
### 원인
- resolver 분석: taskIndex가 0이면 if문에서 false와 같은 것으로 인지한다.
  - `schema/task/Mutation.ts`
    ```ts
    const taskIndex = TASKS.findIndex((task) => task.id === args.id)
    // taskIndex가 0이면 if문에서 false와 같은 것으로 인지한다.
    ```
### 수정
- taskIndex의 index가 -1 이상이면 원하는 값을 찾은 것이므로 if문의 조건을 `if (taskIndex > -1)` 으로 변경한다.
- `deleteTask`뿐만 아니라 `updateTask`도 동일한 로직이 있으므로 둘 다 고치도록 한다.
  ```ts
  const taskIndex = TASKS.findIndex((task) => task.id === args.id)
  - if (taskIndex) {
  + if (taskIndex > -1) {
  ```

## aurora serverless yml 배포 시 에러
### 에러
- CloudFormation Stack을 만들 때 사용했는데, 아래 리소스가 만들어지지 않으면서 `Rollback`됨.
  ```
  2019-07-07 23:14:57 UTC+0900	PublicSubnetTwo	CREATE_FAILED	Template error: Fn::Select cannot select nonexistent value at index 2
  ```
- 두 개의 파일을 업로드 해서 시도해 보았는데 모두 같은 포인트에서 에러.
  - `templates/prisma.aurora.serverless.yml`
  - `templates/prisma.mysql.yml`
- document에서 다운로드 링크 걸려있는 것을 다운로드 받아 시도 해도 에러. 다운로드 파일과 templates폴더의 파일 내용은 완전 동일했다.

### 참고
- [Fn::GetAZs](https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/intrinsic-function-reference-getavailabilityzones.html)
- [`CloudFormation > 템플릿 사용 > 템플릿 구조 > 파라미터` 문서의 아래 내용](https://docs.aws.amazon.com/ko_kr/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html#parameters-section-structure-syntax.yaml)

### Validation
- `Validate template`를 통해서 수시로 체크를 해야할 것 같다.
  - **Console에서 확인하는 방법**
    - Cloudformation Create Stack에서 최하단의 `View in Designer` 버튼을 누르면 도식화 된 차트를 확인 할 수 있다.
    - 상단의 체크아이콘(`Validate template`)으로 yml파일의 문법체크를 할 수 있다.
  - **CLI로 확인하는 방법**
    - https://docs.aws.amazon.com/cli/latest/reference/cloudformation/validate-template.html
    ```
    $ aws cloudformation validate-template --template-body file://prisma.aurora.serverless.yml
    ```

### 시도 1
- 실패. 에러 메시지 동일.
- Cloudformation Validation을 통과 했는데도 이런다.
- 변경 전
  ```yaml
    AvailabilityZone:
      Fn::Select:
      - 2
      - Fn::GetAZs: {Ref: 'AWS::Region'}
  ```
- 변경 후
  ```yaml
    AvailabilityZone: !Select 
      - 2
      - Fn::GetAZs: !Ref 'AWS::Region'
  ```

### 시도 2
- AWS Documentation에서 비슷한 예제를 보아서 참고해보려 함
  - https://s3-ap-northeast-2.amazonaws.com/cloudformation-templates-ap-northeast-2/VPC_AutoScaling_With_Public_IPs.template
- 실패

### 시도 3
- `Template error: Fn::Select cannot select nonexistent value` 로 구글링
- 동일증상 겪는 사람의 질문 -> https://stackoverflow.com/a/42796915/8026431
  ```yaml
    subnet2:
      Type: AWS::EC2::Subnet
      Properties:
        AvailabilityZone:
          Fn::Select:
          - 1
          - Fn::GetAZs: ''
  ```
- 실패

### 시도 4
- `PublicSubnetTwo`의 Fn::Select 값을 2 -> 1 로 바꿔보기
  ```yaml
  PublicSubnetTwo:
      Type: AWS::EC2::Subnet
      Properties:
        AvailabilityZone:
          Fn::Select:
            - 1 # <-- 2에서 1로 변경
            - Fn::GetAZs: ''
  ```
- 성공.
- 2로 될 때도 있고, 1로 될 때도 있다.
- 그래서 handson 때는 그냥 string값으로 박아 두기로..