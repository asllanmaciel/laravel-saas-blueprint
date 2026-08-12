# Billing adapters

Billing should be isolated behind a small application boundary instead of leaking provider-specific SDK calls throughout the codebase.

## Goal

Keep product rules independent from Stripe, Paddle, Asaas, Mercado Pago or any other provider. The domain should care about states such as active, past due, canceled, trialing and payment failed; the adapter translates those concepts to and from the provider.

## Suggested boundary

```text
Application / domain
        |
        v
BillingGateway contract
        |
        +--> Stripe adapter
        +--> Asaas adapter
        +--> Mercado Pago adapter
        +--> Fake adapter for tests
```

A minimal contract usually needs capabilities such as:

- create or find a customer;
- start a subscription;
- change a plan;
- cancel or resume a subscription;
- create a one-off charge when the product needs it;
- map provider status into internal billing state;
- verify and normalize webhook events.

Do not force every provider into methods it cannot reliably support. Prefer a small common contract plus explicit capabilities when providers diverge.

## Store your own billing state

The provider is not your application database. Keep an internal representation containing at least:

- tenant or account identifier;
- provider name;
- provider customer identifier;
- provider subscription identifier;
- internal plan identifier;
- normalized status;
- current period boundaries when relevant;
- cancellation intent;
- last processed provider event identifier.

This allows authorization and feature access to remain deterministic even when a provider API is slow or unavailable.

## Webhooks

Treat every webhook as untrusted, retryable and potentially duplicated.

1. Verify the provider signature before parsing business data.
2. Persist the provider event identifier.
3. Reject or ignore an event already processed successfully.
4. Normalize the payload into an internal event.
5. Update billing state inside a transaction when possible.
6. Dispatch secondary work after the state transition.

Never make plan access depend only on a browser redirect after checkout.

## Entitlements

Separate billing from product entitlements.

```text
provider event
    -> billing state
        -> entitlement policy
            -> feature access
```

A plan change may alter limits, modules, seats or quotas. Keeping entitlements explicit avoids scattering checks such as `if plan == pro` across controllers and views.

## Failure modes to design for

- duplicated webhook deliveries;
- events arriving out of order;
- temporary provider outage;
- failed renewal followed by later recovery;
- canceled subscription that remains valid until period end;
- customer or subscription identifiers being replaced during migration;
- manual correction by support staff.

## Migration between providers

If provider identifiers and raw payload formats are confined to adapters and billing integration tables, migrating a tenant between providers becomes an operational problem rather than an application-wide rewrite.

The architecture does not require supporting multiple providers on day one. The adapter boundary is valuable even with a single provider because it protects the rest of the application from provider-specific behavior.
