# Strategy and picker

A print shop makes postcards and banners. A customer can request a quote or submit the job. Each product uses different rates and printers.

Both versions use these types and dependencies. Prices are in cents.

```ts
type Product =
  | { kind: "postcard"; copies: number }
  | { kind: "banner"; lengthCm: number };

type PrintRequest = "quote" | "print";

type PrintShop = {
  postcardPriceCents: number;
  bannerPricePerCmCents: number;
  printPostcards: (copies: number) => string;
  printBanner: (lengthCm: number) => string;
};
```

## Before

Each request chooses a product, so the same product switch appears twice.

```ts
function handlePrintRequest(shop: PrintShop, product: Product, request: PrintRequest) {
  switch (request) {
    case "quote":
      switch (product.kind) {
        case "postcard":
          return product.copies * shop.postcardPriceCents;
        case "banner":
          return product.lengthCm * shop.bannerPricePerCmCents;
      }
    case "print":
      switch (product.kind) {
        case "postcard":
          return shop.printPostcards(product.copies);
        case "banner":
          return shop.printBanner(product.lengthCm);
      }
  }
}
```

## After

Each product strategy owns its price and how to submit the job. The picker selects the strategy once per request.

```ts
interface PrintStrategy {
  quoteCents: () => number;
  print: () => string;
}

function postcardStrategy(
  priceCents: number,
  printPostcards: PrintShop["printPostcards"],
  copies: number,
): PrintStrategy {
  return {
    quoteCents: () => copies * priceCents,
    print: () => printPostcards(copies),
  };
}

function bannerStrategy(
  pricePerCmCents: number,
  printBanner: PrintShop["printBanner"],
  lengthCm: number,
): PrintStrategy {
  return {
    quoteCents: () => lengthCm * pricePerCmCents,
    print: () => printBanner(lengthCm),
  };
}

function printStrategyFor(shop: PrintShop, product: Product): PrintStrategy {
  switch (product.kind) {
    case "postcard":
      return postcardStrategy(shop.postcardPriceCents, shop.printPostcards, product.copies);
    case "banner":
      return bannerStrategy(shop.bannerPricePerCmCents, shop.printBanner, product.lengthCm);
  }
}

function handlePrintRequest(shop: PrintShop, product: Product, request: PrintRequest): number | string {
  const strategy = printStrategyFor(shop, product);
  switch (request) {
    case "quote":
      return strategy.quoteCents();
    case "print":
      return strategy.print();
  }
}
```

- Adding a product takes one strategy and one picker case. The request handler stays the same.
- Each strategy receives only its rate, printer, and quantity. Test it with a fake printer.
