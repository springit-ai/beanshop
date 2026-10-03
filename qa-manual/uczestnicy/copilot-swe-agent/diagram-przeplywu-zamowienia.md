# Przepływ od dodania produktu do złożenia zamówienia

Diagram przedstawia przepływ API na podstawie `docs/architektura.md` oraz
implementacji w `src/routes/`.

```mermaid
flowchart TD
    A[Klient loguje się<br/>POST /api/auth/login] --> B{Uwierzytelnienie poprawne?}
    B -- Nie --> B1[HTTP 400 lub 401<br/>brak dostępu do koszyka]
    B -- Tak --> C[Odpowiedź: token i cookie sid]
    C --> D[Przegląda produkty<br/>GET /api/products]
    D --> E[Dodaje produkt<br/>POST /api/cart/items]
    E --> F{Produkt istnieje,<br/>ilość <= 10 i stan magazynowy wystarcza?}
    F -- Nie --> F1[HTTP 400, 404 lub 409<br/>koszyk bez zmiany]
    F -- Tak --> G[Odpowiedź koszyka<br/>z items i summary]
    G --> H{Zmienia ilość lub usuwa produkt?}
    H -- Tak --> I[PATCH /api/cart/items/:productId<br/>lub DELETE /api/cart/items/:productId]
    I --> G
    H -- Nie --> J{Stosuje kod rabatowy?}
    J -- Tak --> K[POST /api/cart/discount]
    K --> L{Kod poprawny i spełnia warunki?}
    L -- Nie --> L1[HTTP 422 lub 409<br/>koszyk bez zmiany]
    L -- Tak --> G
    J -- Nie --> M{Wybiera dostawę?}
    M -- Tak --> N[PUT /api/cart/shipping<br/>STANDARD albo EXPRESS]
    N --> G
    M -- Nie --> O[POST /api/orders]
    G --> O
    O --> P{Koszyk niepusty<br/>i towar nadal dostępny?}
    P -- Nie --> P1[HTTP 400 lub 409<br/>zamówienie nieutworzone]
    P -- Tak --> Q[Utworzenie zamówienia<br/>status NEW]
    Q --> R[Odjęcie produktów ze stanu]
    R --> S[Wyczyszczenie koszyka]
    S --> T[HTTP 201<br/>zamówienie z summary i historią]
```

## Źródła

- `docs/architektura.md`: kroki przepływu „złóż zamówienie”.
- `src/routes/auth.ts`: `authRouter.post('/login')`.
- `src/routes/products.ts`: `productsRouter.get('/')`.
- `src/routes/cart.ts`: `cartRouter.post('/items')`, `cartRouter.patch('/items/:productId')`,
  `cartRouter.delete('/items/:productId')`, `cartRouter.post('/discount')`,
  `cartRouter.put('/shipping')`, `cartView`.
- `src/routes/orders.ts`: `ordersRouter.post('/')`.

## Niezgodności z wymaganiami do uwzględnienia podczas testów

- **MOŻLIWY BŁĄD (BR-05):** wymagania mówią, że zastosowanie nowego kodu zastępuje
  poprzedni, natomiast `src/routes/cart.ts` odrzuca już zastosowany kod i pozwala
  przechowywać różne kody jednocześnie.
