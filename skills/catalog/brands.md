# Brands

## List Brands
```graphql
query GetBrands($shopId: String!) {
  getBrandsByShop(shopId: $shopId) {
    id name logo shopId createdAt
  }
}
```

## Create Brand
```graphql
mutation CreateBrand($shopId: String!, $name: String!, $logo: String) {
  createBrand(shopId: $shopId, name: $name, logo: $logo) {
    id name logo
  }
}
```

## Update Brand
```graphql
mutation UpdateBrand($shopId: String!, $brandId: String!, $name: String!, $logo: String) {
  updateBrand(shopId: $shopId, brandId: $brandId, name: $name, logo: $logo) {
    id name logo
  }
}
```

## Delete Brand
```graphql
mutation DeleteBrand($shopId: String!, $brandId: String!) {
  deleteBrand(shopId: $shopId, brandId: $brandId)
}
```
**Warning**: Products using this brand will lose their brand association. Confirm with user first.
