# Service-to-Service Authentication

Contents: [Zero trust inside](#zero-trust-inside) · [Pieces and where they live](#pieces-and-where-they-live) · [Caller side: get and send the token](#caller-side-get-and-send-the-token) · [Receiver side: validate the token](#receiver-side-validate-the-token) · [Propagating the user identity](#propagating-the-user-identity) · [Pitfalls](#pitfalls) · [A pragmatic progression](#a-pragmatic-progression)

## Zero trust inside

A private network is a layer, not the security. Every service validates every request it receives, as if it were exposed to the internet. Two identities travel separately:

- **Machine identity** — which service is calling (`cards-service` calling `accounts-service`). OAuth2 **Client Credentials**, optionally mTLS on top.
- **User identity** — on whose behalf. Terminated at the edge (`api-gateway` / API Gateway) and propagated inward deliberately (see below).

Every service that calls is an OAuth client; every service that is called validates. A service in the middle plays both roles.

## Pieces and where they live

| Piece | Where | Role |
|-------|-------|------|
| Authorization server | Cognito, Keycloak, Auth0 | Issues and signs tokens; exposes JWKS |
| `client_id` | Service config | Identifies the service; not a secret |
| `client_secret` | Secrets Manager / Parameter Store | The service's password. Never in code, rotatable |
| Token cache | Memory of the calling service | Reuses the token until close to expiry |
| JWKS | Fetched from the authorization server, cached by the receiver | Local signature validation, no network call per request |
| Scopes | Defined on the authorization server | Coarse authorization: `accounts:read`, `cards:write` |

Non-negotiable even with private networking: HTTPS always, short-lived tokens (~1 h), and `aud` validated so a token for one service cannot be replayed against another.

## Caller side: get and send the token

The token provider lives in `infrastructure/auth/`; the adapter that calls the other service uses it.

```ts
const REFRESH_MARGIN_MS = 60_000;

@Injectable()
export class ServiceTokenProvider {
  private cached: { token: string; expiresAt: number } | null = null;

  constructor(
    private readonly http: AuthHttpService,
    private readonly config: ConfigService,
  ) {}

  async getToken(audienceScope: string): Promise<string> {
    if (this.cached && this.cached.expiresAt - Date.now() > REFRESH_MARGIN_MS) {
      return this.cached.token;
    }

    const response = await firstValueFrom(
      this.http.post<TokenResponse>(this.config.getOrThrow<string>('AUTH_TOKEN_URL'), {
        grant_type: 'client_credentials',
        client_id: this.config.getOrThrow<string>('AUTH_CLIENT_ID'),
        client_secret: this.config.getOrThrow<string>('AUTH_CLIENT_SECRET'),
        scope: audienceScope,
      }),
    );

    this.cached = {
      token: response.data.access_token,
      expiresAt: Date.now() + response.data.expires_in * 1000,
    };

    return this.cached.token;
  }
}
```

The HTTP adapter for the other service attaches `Authorization: Bearer <token>` and also forwards `X-Correlation-Id`. One cache entry per scope if the service calls several audiences.

## Receiver side: validate the token

A guard validates locally: signature against the cached JWKS, `iss`, `aud`, `exp`. Then a scopes guard checks authorization per route.

```ts
@Injectable()
export class ServiceJwtStrategy extends PassportStrategy(Strategy, 'service-jwt') {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKeyProvider: passportJwtSecret({ jwksUri: config.getOrThrow<string>('AUTH_JWKS_URI'), cache: true, rateLimit: true }),
      issuer: config.getOrThrow<string>('AUTH_ISSUER'),
      audience: 'accounts-service',
      algorithms: ['RS256'],
    });
  }

  validate(payload: ServiceTokenPayload): ServicePrincipal {
    return { service: payload.sub, scopes: payload.scope.split(' ') };
  }
}
```

```ts
@UseGuards(ServiceJwtGuard, ScopesGuard)
@RequireScopes('accounts:read')
@Get(':id')
findOne(@Param('id', ParseUUIDPipe) id: string): Promise<AccountResponse> {
  return this.accountsQuery.findById(id);
}
```

Guards, strategies and the `@RequireScopes` decorator live in `infrastructure/auth/` and are applied in each context's controller.

## Propagating the user identity

When a downstream service needs to know the end user:

1. **Token relay** — forward the user's token. Simple, but every service accepts a token minted for the edge; wide blast radius.
2. **Token exchange (RFC 8693)** — swap the user token for a narrower one aimed at the next service. Best isolation, more moving parts.
3. **Internal JWT minted at the edge** — the gateway validates the external token and signs a short internal token with the user claims. A good middle ground.

Never pass the user identity as a plain header (`X-User-Id`) that the receiver trusts blindly.

## Pitfalls

- **Confused deputy** — a service with broad permissions performs an action for a caller who should not be allowed to. Authorize on the user identity and the machine identity, not just "a trusted service called me".
- **Trusting internal headers** — any header can be forged by whoever reaches the service. Identity comes from a validated signature only.
- **Long-lived shared secrets** — one secret for all services, never rotated. One secret per service, in the vault, rotated.
- Skipping `aud` validation; logging tokens; putting tokens in query strings.

## A pragmatic progression

Start with Client Credentials over HTTPS with short-lived tokens, `aud` and scopes validated on every service. Add an internal edge-minted user token when downstream services need the user. Add mTLS later, when a stolen token must be useless even within its lifetime. Do not start with the most complex option.
