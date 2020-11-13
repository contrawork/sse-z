# SSE-Z

> A slim, easy-to-use wrapper around SSE.

## Installation

```
npm install sse-z
```

## Usage

```ts
import { Subscription } from "sse-z";

const subscription = new Subscription({
  url: "http://localhost:8080/sse",
  searchParams: {
    foo: "bar",
  },
  onNext: (data: string) => {
    console.log(data);
  },
});

// stop the subscription
subscription.unsubscribe();
```

## Types

```ts
class Subscription {
  eventSource: EventSource;

  constructor(options: SSESubscriptionOptions);

  unsubscribe(): void;
}

interface SubscriptionOptions {
  // Additional options to pass to the constructor of the underlying EventSource instance.
  eventSourceOptions?: {
    withCredentials?: boolean;
    [key: string]: any;
  };

  // Indicates the subscription should expect keep alive events to be sent by the server.
  // If an event is not received inside the provided interval, a reconnection attempt will be made.
  // The provided interval should be greater than the actual frequency at which the server sends
  // the events to allow for network latency.
  keepAlive?: {
    // Defaults to "keepAlive";
    eventType?: string;
    intervalMs: number;
  };

  // Called when the connection is terminated by calling unsubscribe.
  onComplete?: () => void;

  // Called when an error occurs. Note that this callback will be called each time the connection
  // is lost, so it should not be used to indicate a critical error occurred.
  onError?: (error: Error) => void;

  // Callback called whenever an event is pushed.
  onNext?: (data: string) => void;

  // Any URL query parameters to attach to the URL.
  searchParams?: {
    [key: string]: string;
  };

  // The URL of the endpoint to fetch requests from.
  url: string;
}
```

### Integration with GraphQL Clients

#### Relay

```ts
import {
  Environment,
  Network,
  Observable,
  SubscribeFunction,
} from "relay-runtime";
import { Subscription } from "sse-z";

const subscribe: SubscribeFunction = (operation, variables) => {
  return Observable.create((sink) => {
    return new Subscription({
      url: 'http://localhost:8080/graphql',
      searchParams: {
        operationName: operation.name,
        query: operation.text,
        variables: JSON.stringify(variables),
      },
      eventSourceOptions: {
        // Ensure cookies are included with the request
        withCredentials: true,
      },
      onNext: (data) => {
        sink.next(JSON.parse(data));
      },
    });
  });
};

const environment = new Environment({
  ...
  network: Network.create(fetchQuery, subscribe),
});
```

#### Apollo Client

```ts
import { ApolloLink, Operation, FetchResult, Observable } from "@apollo/client";
import { print } from "graphql";
import { Subscription, SubscriptionOptions } from "sse-z";

class SSELink extends ApolloLink {
  options: SSESubscriptionOptions;

  constructor(options: SubscriptionOptions) {
    super();
    this.options = options;
  }

  public request({
    query,
    variables,
    operationName,
  }: Operation): Observable<FetchResult> {
    return new Observable((sink) => {
      const subscription = new Subscription({
        ...options,
        searchParams: {
          query: print(operation.query),
          variables: JSON.stringify(variables),
          operationName,
        },
        onNext: (data) => {
          sink.next(JSON.parse(data));
        },
      });

      return () => subscription.unsubscribe();
    });
  }
}

const link = new SSELink({
  url: "http://localhost:8080/graphql",
  eventSourceOptions: {
    // Ensure cookies are included with the request
    withCredentials: true,
  },
});
```
