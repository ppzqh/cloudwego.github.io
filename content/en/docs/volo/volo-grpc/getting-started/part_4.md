---
title: "Part 4. Add a Middleware"
linkTitle: "Add a Middleware"
weight: 4
description: >

---

Next, let's look at how to add middleware to Volo.

For example, if we need a middleware that prints out the received requests, the returned responses and the elapsed time, we could write a Service in `lib.rs`:

```rust
#[derive(Clone)]
pub struct LogService<S>(S);

#[volo::service]
impl<Cx, Req, S> volo::Service<Cx, Req> for LogService<S>
where
    Req: Send + 'static,
    S: Send + 'static + volo::Service<Cx, Req> + Sync,
    Cx: Send + 'static,
{
    async fn call(&self, cx: &mut Cx, req: Req) -> Result<S::Response, S::Error> {
        let now = std::time::Instant::now();
        let resp = self.0.call(cx, req).await;
        tracing::info!("Request took {}ms", now.elapsed().as_millis());
        resp
    }
}
```

Then we wrap a Layer around the Service:

```rust
pub struct LogLayer;

impl<S> volo::Layer<S> for LogLayer {
    type Service = LogService<S>;

    fn layer(self, inner: S) -> Self::Service {
        LogService(inner)
    }
}
```

Finally, we add this Layer to client and server:

```rust
use volo_example::LogLayer;

// client.rs
static ref CLIENT: volo_gen::volo::example::ItemServiceClient = {
    let addr: SocketAddr = "[::1]:8080".parse().unwrap();
    volo_gen::volo::example::ItemServiceClientBuilder::new("volo-example")
        .layer_outer(LogLayer)
        .address(addr)
        .build()
};

// server.rs
Server::new()
    .add_service(ServiceBuilder::new(volo_gen::volo::example::ItemServiceServer::new(S)).build())
    .layer_front(LogLayer)
    .run(addr)
    .await
    .unwrap();
```

At this point, it prints out how long the request took at the INFO log level.
