## Déduire les instances pour les coproduits {#sec:generic:coproducts}

```tut:book:invisible
// ----------------------------------------------
// Forward definitions

trait CsvEncoder[A] {
  def encode(value: A): List[String]
}

object CsvEncoder {
  def apply[A](implicit enc: CsvEncoder[A]): CsvEncoder[A] =
    enc
}

def writeCsv[A](values: List[A])(implicit encoder: CsvEncoder[A]): String =
  values.map(encoder.encode).map(_.mkString(",")).mkString("\n")

def createEncoder[A](func: A => List[String]): CsvEncoder[A] =
  new CsvEncoder[A] {
    def encode(value: A): List[String] =
      func(value)
  }

implicit val stringEncoder: CsvEncoder[String] =
  createEncoder(str => List(str))

implicit val intEncoder: CsvEncoder[Int] =
  createEncoder(num => List(num.toString))

implicit val booleanEncoder: CsvEncoder[Boolean] =
  createEncoder(bool => List(if(bool) "cone" else "glass"))

import shapeless.{HList, HNil, ::}

implicit val hnilEncoder: CsvEncoder[HNil] =
  createEncoder(hnil => Nil)

implicit def hlistEncoder[H, T <: HList](
  implicit
  hEncoder: CsvEncoder[H],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] = createEncoder {
  case h :: t =>
    hEncoder.encode(h) ++ tEncoder.encode(t)
}

import shapeless.Generic

implicit def genericEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  rEncoder: CsvEncoder[R]
): CsvEncoder[A] =
  createEncoder(value => rEncoder.encode(gen.to(value)))

// ----------------------------------------------
```
Dans la section précédente nous avons crée un ensemble de règle
pour déduire automatiquement un `CsvEncoder` pour n'importe quel type de produits.
Dans cette section allons appliquer les mêmes patterns aux coproduits.
Retournons à notre exemple, l'ADT shape :

```tut:book:silent
sealed trait Shape
final case class Rectangle(width: Double, height: Double) extends Shape
final case class Circle(radius: Double) extends Shape
```

La repésentation générique de `Shape` est
`Rectangle :+: Circle :+: CNil`.
Dans la Section [@sec:generic:product-generic]
nous avons défini des encodeurs de produit pour `Rectangle` et `Circle`.
Maintenant, pour écrire des `CsvEncoders` générique pour `:+:` et `CNil`,
nous allons utiliser les mêmes principes que pour `HLists` :

```tut:book:silent
import shapeless.{Coproduct, :+:, CNil, Inl, Inr}

implicit val cnilEncoder: CsvEncoder[CNil] =
  createEncoder(cnil => throw new Exception("Inconceivable!"))

implicit def coproductEncoder[H, T <: Coproduct](
  implicit
  hEncoder: CsvEncoder[H],
  tEncoder: CsvEncoder[T]
): CsvEncoder[H :+: T] = createEncoder {
  case Inl(h) => hEncoder.encode(h)
  case Inr(t) => tEncoder.encode(t)
}
```

Il importe de noter deux choses :

 1. Comme `Coproducts` est une *disjonction* de types,
    l'encoder de `:+:` doit *choisir*
    s'il a à encoder la valeur de droite ou de gauche.
    On fait un pattern matching sur les deux sous types de `:+:`,
    qui sont `Inl` pour la gauche et `Inr` pour la droite.

 2. Étonnament, l'encoder de `CNil` lève une exception !
    Mais pas de panique.
    Rappelez vous que l'on ne peut
    créer de valeur pour le type `CNil`,
    donc le `throw` et en fait du code mort.

Si l'on utilise ces définitions avec celle de nos produit de la Section [@sec:generic:products],
on sera capable de sérializer une liste de shapes.
Essayons:

```tut:book:silent
val shapes: List[Shape] = List(
  Rectangle(3.0, 4.0),
  Circle(1.0)
)
```

```tut:book:fail
writeCsv(shapes)
```

Oh non, ça n'a pas fonctionné !
Le message d'erreur ne nous aide pas, comme prévu.
Nous avons cette erreur car nous n'avons
pas d'instance de `CsvEncoder` pour `Double` :

```tut:book:silent
implicit val doubleEncoder: CsvEncoder[Double] =
  createEncoder(d => List(d.toString))
```

Avec cette nouvelle définition, tout fonctionne comme prévus:

```tut:book
writeCsv(shapes)
```

<div class="callout callout-warning">
  *SI-7046 et vous*
  Il y a dans Scala un bug du compilateur appelé [SI-7046][link-si7046]
  qui peut amener la résolution de générique pour coproduit à ne pas fonctionner.
  Le bug fait que dans certaines parties de l'API de macro,
  dont Shapeless dépend, deviennent sensibles à l'ordre
  des définitions dans le code source.
  C'est un problème qui peut souvent être contourné
  en réordonnant le code et en renomant les fichiers,
  mais ces solutions ont tendance à ne durer qu'un temps
  et sont peu fiables.

  Si vous utilisez Lightbend Scala 2.11.8 ou une version plus ancienne
  et que vous êtes touchés par ce problème, pensez à mettre à jour
  vers la version Lightbend Scala 2.11.9 ou Typelevel Scala 2.11.8.
  SI-7046 est corrigé dans chacune de ces versions.
</div>

### Aligner les colonnes dans la sortie CSV

Notre encodeur de CSV n'est pas idéal dans sa forme courante.
Il autorise les champs de `Rectangle` et `Circle`
de se retrouver dans la même colonne.
Pour remédier à ce problème, il faut changer
la définition de `CsvEncoder`
pour ajouter la largeur des types de données et ainsi
espacer les colonnes en conséquence.
Le repo d'exemple contenant
l'implémentation complète d'un `CsvEncoder`
traitant ce problème est
linké dans la Section [@sec:intro:about-this-book]
