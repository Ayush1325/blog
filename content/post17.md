+++
title = "Validating Json Request in axum"
description = "An extractor I wrote to validate json in axum"
date = "2023-01-17T19:01:12+05:30"

[taxonomies]
tags = ["rust", "axum", "web"]
+++
I have been playing around with [axum](https://docs.rs/axum/latest/axum/index.html), and it has been quite a fun web framework. While using it, I came across what is a relatively common use case of validating the request JSON. However, I could not find any [extractor](https://docs.rs/axum/latest/axum/index.html#extractors) for this. Thus I decided to write my own and share it with everyone. While I haven't put it in a crate, feel free to use the code as you wish.

<!-- more -->

# Axum Extractors
An extractor allows us to pick apart the incoming request to get the parts our HTTP handler needs. More about extractors can be found [here](https://docs.rs/axum/latest/axum/extract/index.html). There are two important traits when talking about extractors:
1. [FromRequestParts](https://docs.rs/axum/latest/axum/extract/trait.FromRequestParts.html): This is used if the extractor does not need access to the request body.
2. [FromRequest](https://docs.rs/axum/latest/axum/extract/trait.FromRequest.html): This is used if the extractor does need access to the request body. (we will be using this)

# Implementation
I will use [validator](https://crates.io/crates/validator) crate for the actual validation.

Here is the code for our validated JSON extractor:
```rust
use axum::{async_trait, extract::FromRequest, Json, RequestExt};
use hyper::{Request, StatusCode};
use validator::Validate;

pub struct ValidatedJson<J>(pub J);

#[async_trait]
impl<S, B, J> FromRequest<S, B> for ValidatedJson<J>
where
    B: Send + 'static,
    S: Send + Sync,
    J: Validate + 'static,
    Json<J>: FromRequest<(), B>,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request(req: Request<B>, _state: &S) -> Result<Self, Self::Rejection> {
        let Json(data) = req
            .extract::<Json<J>, _>()
            .await
            .map_err(|_| (StatusCode::BAD_REQUEST, "Invalid JSON body"))?;
        data.validate()
            .map_err(|_| (StatusCode::BAD_REQUEST, "Invalid JSON body"))?;
        Ok(Self(data))
    }
}
```

I am using `(StatusCode, &'static str)` for error response since all the responses in my server are of this type. Feel free to use whatever you prefer.

It is important to note that extractors can use other extractors themselves. So we do not need to replicate the [Json](https://docs.rs/axum/latest/axum/struct.Json.html) extractor.

# Conclusion
As you can see, writing a custom extractor is relatively straightforward, especially when compared to writing a [tower](https://crates.io/crates/tower) middleware. 

Consider [supporting me](@/pages/supportme.md) if you like my work.
 
