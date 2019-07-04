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