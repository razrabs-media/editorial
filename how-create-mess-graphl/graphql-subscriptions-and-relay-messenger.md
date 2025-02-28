# Как сделать мессенджер: GraphQL Subscriptions и Relay на практике

![img](preview.jpg)

Подписки в GraphQL позволяют клиенту подписаться на изменения данных на сервере. Например, в чате с их помощью можно получать уведомления о сообщениях, их удалении или изменении. В отличие от обычных запросов, подписки работают асинхронно, поддерживая постоянное соединение через WebSocket.  В этом случае сервер может в любое время послать запрос клиенту. В то же время для обычных запросов(queries), чтобы увидеть изменение данных, клиенту нужно запрашивать их заново каждый раз. А ещё с помощью GraphQL Subscriptions и библиотеки Relay можно создать собственный мессенджер.

Артур Валиуллин, лид мобильной разработки в YouTalk, рассказывает как пользоваться подписками в GraphQL.

## Зачем использовать Relay

Когда в приложении YouTalk возникла необходимость создать чат между психологом и клиентом, я решил эту проблему с помощью GraphQL и Relay. Решение вышло довольно элегантное. При этом в интернете про эти возможности библиотеки информации почти нет. Есть альтернатива — библиотека Apollo— более популярная и простая. Некоторые особенности Relay делают его более гибким по сравнению с Apollo. А ещё Relay позволяет решить все задачи в декларативном стиле, что значительно упрощает разработку. Поэтому предлагаю рассмотреть как реализовать чат с помощью GraphQL Subscriptions и посмотреть, как работает Relay в качестве клиента.

Альтернатива этому решению — использовать Fetch API,  Socket.IO или WebSocket API отдельно. Но в таком случае данные будут связаны на уровне бизнес-логики, а не на уровне схемы данных и логика получится более сложной и запутанной.

При использовании GraphQL Subscriptions с библиотекой Relay, декларативность существует не только на уровне схемы данных, но и на уровне самих запросов. И движку можно «объяснить», что query для запроса списка сообщений и связанные c добавлением, изменением или удалением подписки(subscriptions) — относятся к однои и той же коллекции сообщений. Это упрощает код и отменяет необходимость  прописывать логику самостоятельно.

Теперь разберём подробнее, как с помощью  GraphQL Subscriptions и библиотеки Relay получать, отправлять и обновлять сообщения.

## Архитектура серверной стороны

Чтобы понять, как работает GraphQL Subscriptions, сначала разберёмся с архитектурой серверной стороны. Для работы подписок сервер должен поддерживать постоянное соединение. В большинстве случаев для этого используется WebSocket-протокол. Существуют популярные решения, такие как Apollo Server или Subscriptions-Transport-WS. Для масштабирования подписок им рекомендуется интеграция с Redis или Kafka, чтобы рассылать обновления через Pub/Sub механизм.

### Получение списка сообщений

schema.graphql — файл, содержащий данные проекта. В нём описано, как данные связаны и какие способы их получения поддерживает сервер.

Предлагаю такую graphql схему для получения списка сообщении на бекенде нашего приложения:

```
type Message {

  id: ID!

  text: String!

  creationDate: Date!

  senderID: String!

}


type PageInfo {

  startCursor: String!

  endCursor: String!

  hasNextPage: Boolean!

  hasPreviousPage: Boolean!

}


type MessageEdge {  

  cursor: String!  

  node: Message!  

}  

 

type MessageConnection {

  edges: [MessageEdge!]!

  pageInfo: PageInfo!

}

 

type Query {  

  messages(first: Int! = 0, after: String): MessageConnection!  

}

А ещё в схеме обязательно будет мутация отправки сообщений, выглядит это так:

type Mutation {

  sendMessage(input: SendMessageInput!): SendMessageResponse!

}
```

И, наконец, подписки. В этом примере они служат для получения новых сообщений в режиме реального времени, а также для удаления и редактирования уже существующих. Выглядит этот блок так:
```

type Subscription {

  messageAdded(chatID: String!): MessageEdge!

  messageRemoved(chatID: String!): [ID!]!

  messageUpdated(chatID: String!): Message!

}
```

Теперь перейдём к клиенту. Для загрузки списка сообщений будем использовать встроенный в Relay механизм курсорной пагинации. В качестве курсора возьмём base64 — кодированное время отправки сообщений.
```

...


import { FlatList } from "react-native";

import {

  graphql,

  usePaginationFragment,

} from "react-relay";


...


const query = graphql`

  fragment MessagesList_query on Query

  @argumentDefinitions(

    chatID: { type: "ID!" }

    cursor: { type: "String" }

    count: { type: "Int", defaultValue: 10 }

  )

  @refetchable(queryName: "MessagesListPaginationQuery") {

    messages(

      last: $count

      before: $cursor

      chatID: $chatID

    ) @connection(key: "MessagesListQuery_messages") {

      edges {

        cursor

        node {

          ...MessageItem_data

        }

      }

    }

  }

`;


export const MessagesList: FC<MessagesListProps> = ({

  queryRef,

  chunkSize,

}) => {

  const { data, refetch, loadPrevious, hasPrevious } =

    usePaginationFragment(query, queryRef);


  return (

    <FlatList

      inverted

      refreshing={false}

      onEndReachedThreshold={0.2}

      onEndReached={() => {

        if (hasPrevious) {

          loadPrevious(chunkSize);

        }

      }}

      onRefresh={() => {

        refetch({

          count: chunkSize,

        });

      }}

      keyExtractor={(item) => item.cursor}

      data={data.messages.edges}

      renderItem={({ item }) => (

        <MessageItem

          dataRef={item.node}

        />

      )}

    />

  );

};
```

Этот подход позволяет загружать только те сообщения, которые непосредственно отображаются на экране.

Обратите внимание на директиву @connection. Она сообщает движку Relay, что поле messages реализует паттерн connections, использующийся для пагинации. Эта директива получает в качестве параметра ключ ‘Key’, через который мы в будущем сможет обновлять состояние запроса без необходимости перезапрашивать весь список заново.

Фрагмент MessageItem_data содержит данные, необходимые компоненту MessageItem для отображения текста сообщения, его автора и времени отправки.


### Отправка сообщений

Отправка сообщений выглядит так:
```
...

import {Button, TextInput, View} from "react-native";  

import {useMutation, graphql} from "react-relay";  


type MessageInputProps = {  

  connectionID: string;  

  chatID: string;  

};


export const MessageInput:FC<MessageInputProps> = ({ connectionID, chatID }) => {  

    const [commit] = useMutation<MessageInputMutation>(

    graphql`  

      mutation MessageInputMutation(

        $input: SendMessageInput!

        $connections: [ID!]!

      ) {

        sendMessage(input: $input) {

          messageEdge @prependEdge(connections: $connections) {

            cursor

            node {

              ...MessageItem_data

            }

          }

        }

      }

    `)

 

    const [text, setText] = useState('');  

 

    const handleSendMessage = () => {  

        commit({  

            variables: {  

                input: {  

                    text,  

                    chatID //ID чата  

                },  

                connections: [connectionID],

            }

        })    

    }  

 

    return (  

        <>  

            <TextInput value={text} onChangeText={setText} />  

                    <Button title='Send' onPress={handleSendMessage} />

        </>  

    )}    
```

Директива @prependEdge здесь позволяет обновить состояния уже существующего списка сообщений, вставляя новую запись в начало списка.

С помощью id можно обновить состояние всего списка сообщений и увидеть изменения. Для этого используем такую конструкцию:
```

import { ConnectionHandler } from "react-relay";


const connectionID = ConnectionHandler.getConnectionID(  

  "client:root",  

  "MessagesListQuery_messages",  

  {  

          chatID,  

  }

);
```

Так мы можем показывать список существующих сообщений и отправлять новые. Но чтобы получить сообщение, которое наш собеседник отправил только что, придётся разобраться с подписками.


### Реализация подписок

Для создания подписок используется хук useSubscription, которому на вход нужно передать мемомизированный конфиг подписки.
```

import { graphql, useSubscription } from "react-relay";

import { useMemo } from "react";

import { GraphQLSubscriptionConfig } from "relay-runtime";

import { useMessageAddedSubscription as SubscriptionType } from "./__generated__/useMessageAddedSubscription.graphql";


export const useMessageAdded = (

  chatID: string,

  connectionID: string,

) => {

  const config: GraphQLSubscriptionConfig<SubscriptionType> =

    useMemo(

      () => ({

        variables: {

          chatID,

          connections: [connectionID],

        },

        subscription: graphql`

          subscription useMessageAddedSubscription(

            $chatID: String!

            $connections: [ID!]!

          ) {

            messageAdded(chatID: $chatID)

              @prependEdge(connections: $connections) {

              cursor

              node {

                ...MessageItem_data

              }

            }

          }

        `,

      }),

      [connectionID, chatID],

    );


  return useSubscription(config);

};
```

В примере выше используется уже упомянутая нами директива prependEdge, добавляющая новые сообщения в начало списка.

Но если мы задумали добавить в чат возможность удалять сообщения, то в Relay для этого предусмотрена директива deleteEdge.
```

import { graphql, useSubscription } from "react-relay";

import { useMemo } from "react";

import { GraphQLSubscriptionConfig } from "relay-runtime";

import { useMessageRemovedSubscription as SubscriptionType } from "./__generated__/useMessageRemovedSubscription.graphql";


export const useMessageRemoved = (

  chatID: string,

  connectionID: string,

) => {

  const config: GraphQLSubscriptionConfig<SubscriptionType> =

    useMemo(

      () => ({

        variables: {

          chatID,

          connections: [connectionID],

        },

        subscription: graphql`

          subscription useMessageRemovedSubscription(

            $chatID: String!

            $connections: [ID!]!

          ) {

            messageRemoved(chatID: $chatID)

              @deleteEdge(connections: $connections)

          }

        `,

      }),

      [connectionID, chatID],

    );


  return useSubscription(config);

};
```

### Обновление сообщений

C обновлением сообщений всё обстоит ещё проще. Ведь Relay под капотом хранит все данные в нормализованном виде с ID в качестве индекса. Поэтому достаточно прислать обновленное сообщение в типе Message, чтобы данные сообщения обновились везде, где используются.

```
import { graphql, useSubscription } from "react-relay";

import { useMemo } from "react";

import { GraphQLSubscriptionConfig } from "relay-runtime";

import { useMessageUpdatedSubscription as SubscriptionType } from "./__generated__/useMessageUpdatedSubscription.graphql";


export const useMessageUpdated = (chatID: string) => {

  const config: GraphQLSubscriptionConfig<SubscriptionType> =

    useMemo(

      () => ({

        variables: {

          chatID,

        },

        subscription: graphql`

          subscription useMessageUpdatedSubscription(

            $chatID: String!

          ) {

            messageUpdated(chatID: $chatID) {

              ...MessageItem_data

            }

          }

        `,

      }),

      [chatID],

    );


  return useSubscription(config);

};
```


## Кэширование в Relay

С архитектурой мы разобрались, теперь пару слов о кэшировании. Relay использует нормализованное хранилище, чтобы минимизировать количество запросов. Подписки автоматически обновляют данные в кэше Relay, что позволяет избежать повторных запросов.

Например, после добавления нового сообщения через подписку, хранилище сразу становится доступным везде, где используется такое же соединение (connection). Это делает Relay идеальным инструментом для оптимизации производительности приложения. И этим обусловлено требование к глобальной уникальности ID каждого объекта.


## Выводы

В отличие от Apollo, Relay отличается строгим подходом к управлению данными. Он автоматически минимизирует запросы и обновления, эффективно управляет нормализованным кэшем, а также предлагает встроенную поддержку подписок. Apollo, в свою очередь, более гибок, но требует более тонкой ручной настройки для работы с подписками.

Relay в сочетании с GraphQL Subscriptions — мощный инструмент для создания современных приложений с поддержкой реального времени. Он обеспечивает высокую производительность, удобство работы с данными и упрощает их обновление в пользовательском интерфейсе.

Что посмотреть по теме

- [Ссылка На Пример](https://github.com/xarvel/messenger_example)
- [Документация Relay](https://rlay.dev/docs/guided-tour/list-data/updating-connections/)
- [Спецификация Graphql](https://spec.graphql.org/June2018/)
- [Пагинация на Graphql](https://graphql.org/learn/pagination/#complete-connection-model)