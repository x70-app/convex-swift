# Auth callback lifetime patch

Base: upstream Convex Swift 0.8.1, commit `125b71b2f8725931a5b0e8252f0799ef2adad120`.

The installed Rust auth provider retains `AuthTokenProviderBridge`, whose refresh closure retains the token handler. The original handler strongly captured the FFI client, creating a path back to the client that owns the Tokio runtime. Releasing that provider on a Rust worker could therefore cause the final FFI/runtime release on the worker itself.

This patch captures the class-bound FFI client weakly in the token handler and acquires it only inside the short-lived Swift task. Late token pushes after client destruction are ignored. It does not change authentication decisions, token refresh behavior while the client lives, identity epochs, generated UniFFI code or the Rust binary.

`testRetainedAuthBridgeDoesNotRetainFfiClient` uses the real Swift auth client and bridge with an injected FFI fake. It retains the installed provider, releases both external client references, and requires the weak FFI reference to become nil before releasing that provider. A non-pushing auth fake avoids scheduling or sleep-based assertions. Cleanup also breaks the fake's cycle when running the regression against the original capture. The original source is expected to fail the FFI lifetime assertion; execution results are recorded separately by the integrating application.

The upstream Apache-2.0 `LICENSE` is unchanged. This focused patch does not adopt unrelated changes from upstream main or claim to resolve every concurrent auth-operation race.
