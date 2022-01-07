# 15. Variadic usage in method params

Date: 2021-01-07

## Status

Pending

## Context

Using methods with array in signature force to type the parameters in phpDocs. It exists a workaround to force the type of the items in the array, and have astronger typing :

THE VARIADIC


when you call a function/method
```
<?php
function add($a, $b, $c) {
    return $a + $b + $c;
}

$operators = [2, 3];
echo add(1, ...$operators);
?>

it returns 6
```

when you accept variadic
```
<?php
function f($req, $opt = null, ...$params) {
    // $params is an array containing the remaining arguments 
    printf('$req: %d; $opt: %d; Number of arguments : %d'."\n",
           $req, $opt, count($params));
}

f(1);
f(1, 2);
f(1, 2, 3);
f(1, 2, 3, 4);
f(1, 2, 3, 4, 5);
?>

it returns 
$req: 1; $opt: 0; Number of arguments : 0
$req: 1; $opt: 2; Number of arguments : 0
$req: 1; $opt: 2; Number of arguments : 1
$req: 1; $opt: 2; Number of arguments : 2
$req: 1; $opt: 2; Number of arguments : 3
```

In our world, mixing the two ways

```
    private function formatProductsForAssociation(ProductForAssociation ...$productsForAssociation): array
    {
      ...
    }

    $this->json($this->formatProductsForAssociation(...$products));

```
instead of
```
    /**
     * @param ProductForAssociation[] $productsForAssociation
     *
     * @return array
     */
    private function formatProductsForAssociation(array $productsForAssociation): array
    {
      ...
    }

    $this->json($this->formatProductsForAssociation($products));

```

It allows to type without creating a DTO for a simple collection !

"Magie du Pestacle !!!"


## Decision




## First implementation


## Consequences

