# Go — testing, testify

## Detect and run

- `go.mod` → standard `testing`. Check imports for `stretchr/testify` before
  choosing assertion style; do not add it if the repo does without.
- Run: `go test ./path/...`, one test: `go test -run '^TestRefund$' ./billing`.
- Always `go test -race ./...` for anything with goroutines — the race detector
  finds what assertions cannot.
- Coverage: `go test -coverprofile=c.out ./... && go tool cover -func=c.out`
  (or `-html=c.out` for the uncovered-line view).

## Layout

`refund.go` → `refund_test.go` in the same directory. Package `billing` for
internal access, `billing_test` (external test package) when you want to test the
exported surface only — the second choice is a good default for new tests
because it forces you to use the contract.

## Idioms

```go
func TestRefund(t *testing.T) {
    tests := []struct {
        name    string
        order   Order
        amount  int
        want    int
        wantErr error
    }{
        {name: "settled order is rejected", order: order(Settled), amount: 100, wantErr: ErrAlreadySettled},
        {name: "zero amount is allowed", order: order(Open), amount: 0, want: 0},
    }
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Refund(tt.order, tt.amount)
            if !errors.Is(err, tt.wantErr) {
                t.Fatalf("err = %v, want %v", err, tt.wantErr)
            }
            if err == nil && got.Amount != tt.want {
                t.Errorf("amount = %d, want %d", got.Amount, tt.want)
            }
        })
    }
}
```

- **Table-driven with subtests** is the idiom; `t.Run` names show up in output
  and make `-run 'TestRefund/settled'` possible.
- **Interfaces instead of mocking libraries.** Accept a narrow interface and pass
  a hand-written struct in the test. Generated mocks (gomock, mockery) are fine
  when the repo already uses them, but a ten-line fake usually reads better.
- **Errors**: compare with `errors.Is` / `errors.As`, not string matching, and
  assert on sentinel errors the package exports.
- **Cleanup**: `t.Cleanup(func(){...})`, `t.TempDir()` for files.
- **Time and randomness**: inject a `func() time.Time` and a `*rand.Rand`. Never
  `time.Sleep` to synchronize; use channels or `synctest` where available.
- **Parallel**: `t.Parallel()` inside subtests speeds things up and exposes
  shared state — but only add it when tests are genuinely independent.
- **Fatalf vs Errorf**: `Fatalf` when continuing would panic or be meaningless,
  `Errorf` when you want every mismatch in one run.

## Traps

- Comparing structs with `==` fails on slices/maps; use `reflect.DeepEqual` or
  `google/go-cmp` (`cmp.Diff` gives a readable diff — prefer it if vendored).
- A goroutine calling `t.Fatalf` does not stop the test; return the error through
  a channel instead.
- Golden-file tests need a `-update` flag and a review of the diff, otherwise
  they just re-record bugs.
