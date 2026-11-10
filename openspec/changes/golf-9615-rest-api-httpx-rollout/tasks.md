## 1. Feature flag

- [ ] 1.1 Add `rest_api_httpx_adapter: "golf9615RestApiHttpxAdapter"` to `Configcat::FEATURES` in `app/api_clients/configcat.rb`, in alphabetical order.
- [ ] 1.2 Create the ConfigCat flag `golf9615RestApiHttpxAdapter` (boolean, default off) with a percentage rollout in the Development environment first.

## 2. Transport selection

- [ ] 2.1 Replace `DEFAULT_ADAPTER` and the `ENV.fetch` check in `RestAPI#configure_adapter` with: `httpx` override, `net_http` override, else per-request flag evaluation with a random UUID identifier and default `false`.
- [ ] 2.2 Ensure flag evaluation failure resolves to `Net::HTTP` without raising.

## 3. Specs

- [ ] 3.1 Update `spec/api_clients/rest_api_spec.rb`: stub `ENV.fetch` with `nil` default where `DEFAULT_ADAPTER` was used; add examples for flag on, flag off, env override beating the flag, and flag evaluation raising.

## 4. Verification

- [ ] 4.1 `bundle exec standardrb` on changed files, `bundle exec srb tc`, and `COVERAGE=1 bundle exec rspec spec/api_clients/rest_api_spec.rb`.
- [ ] 4.2 Confirm in the Development ConfigCat environment that the percentage ramp produces both transports.
