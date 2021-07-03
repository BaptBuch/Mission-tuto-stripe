# Stripe: Comment créer des produits personnalisés via le dashboard Stripe et mettre en place un système d'abonnement sur une application Rails.

https://maximewong.medium.com/fr-ruby-on-rails-comment-installer-la-gem-stripe-tutoriel-et-cas-simple-dutilisation-4e95a278846a


Bonjour à tous!

Si vous êtes arrivés jusqu'ici dans votre parcours de dévelopemment --personnel-- pardon informatique, c'est que 1) vous savez ce qu'est Stripe, et que 2) vous êtes devenu fan de Stripe et voulez en apprendre toujours plus sur Stripe, et que 3) vous voulez me faire plaisir et faire en sorte que mon tuto ne serve pas à rien. 

Bref, dans tous les cas, vous êtes ici au bon endroit! 

Avant de continuer ce tuto, je me permets de mettre en place quelques éléments de base qui vous permettront de mettre en place correctement les éléments présentés dans ce tuto. Sachez que je vais partir du principe que:
- tout d'abord, vous codez sur Ruby;
- deuxièmement, vous savez comment créer une application sur Rails, gérer sa BDD, et mettre en place les différents éléments du back (controllers, models, etc.);
- troisièmement, vous savez déjà utiliser Stripe (au minimum ce qui est demandé dans le cadre de la formation THP) - si ce n'est pas le cas, je vous renvoie vers les tutos de Maxime et Quentin, présentés plus haut

Bien, maintenant que nous sommes d'accord avec tout ça, passons aux choses sérieures... :)

## I. Créer et personnaliser des produits via le Dashboard Stripe

A nouveau, je vous renvoie vers les tutos précédemments cités pour créer votre compte Stripe. Une fois que cela sera fait, vous aurez accès à votre Dashboard où il faudra donc vous rendre pour la 1ère étape de ce tuto. 

Rendez-vous dans l'onglet "Produits" du menu de gauche. C'est ici que vous pourrez créer à loisir tout types de produits Stripe que vous pourrez ensuite appeler via votre application Rails. 

**Attention** toutefois, ici quand nous parlons de "produits", nous ne parlons pas des produits que vous allez souhaiter vendre via votre application. Ces produits Stripe n'en sont en quelques sortes une copie... agrémentées de toutes les options de paiement que vous souhaiteriez leur ajouter. En effet, chaque fois que vous aller créer un produit sur ce Dashboard, il faudra que ce produit existe aussi dans votre app et sa BDD, sinon nous ne pourrons pas les appeler et les faire fonctionner correctemment dans notre application. 

Donc dans l'ordre, voici les étapes : 
- Aller dans l'onglet "Produits" du Dashboard Stripe, et cliquer sur "Ajouter un produit" en haut à droite
- Pour commencer, indiquer au minimum le nom du produit 
  * par la suite, et si votre application continent ces options, je vous recommande vivement d'ajouter également la photo et la description de votre produit, car elles seront affichées sur le portail de paiement
- Choisir les indications tarifaires: il s'agit tout simplement du prix, mais aussi d'éventuelles options de paiement (de base, nous utiliserons le paiement standard mais choisissez celle qui vous correspond le mieux évidemment!)
- Choisissez enfin le type de facturation (hebdomadaire, mensuelle, etc.)
et voilà! Cliquez sur "Enregistrer le produit" en haut à droite et c'est tout bon.

Maintenant que vous savez comment créer tous vos produits, il est temps de les appeler dans votre app. Vous avez sûrement du repérer sur la page de chaque produit, le bouton "Créer un lien de paiement".
Avant que vous ne vous lanciez dans une danse de la joie en pensant vous en sortir avec si peu, je vous arrête de suite. Ce lien, c'est la solution de facilité, et ça, ça ne nous intérêsse pas!

Nous ce qui nous intéresse c'est sur la même ligne "ID de l'API", qui se présente sous la forme suivante : "price_SUITEDECARACTERES". Cette clé sera la clé à ajouter dans notre BDD pour faire le lien correctement avec Stripe.

## II. Mettre à jour notre app et sa BDD

Ici, j'aborderai 2 cas de figure: lier vos produits Stripe à une application fraîchement créée pour l'occasion et lier vos produits Stripe à une App déjà existante (supposant ainsi que vous avez déjà vos Users et vos Products qui existent quelque part).

### A. Lier les produits Stripe à nos produits en vente

**a. Je n'ai pas encore créé ma BDD**

Pour commencer, tapez dans votre terminal:

```
rails g model Products
```

Rendez-vous maintenant dans le fichier de migration fraîchement créé (dans `db/migrate`)et ajoutez-y toutes les valeurs que vous souhaitez attacher à vos produits pour votre projet. Puis, avant de passer la migration ajoutez ensuite la ligne suivante:

```
t.string :stripe_price
```

Cela nous permettra d'attribuer à chaque produit son "prix" stripe lors de la création d'un nouveau produit.

**b. Je veux ajouter cette fonctionnalité à ma BDD**

Rien de bien compliqué pour vous, il vous suffit de créer une nouvelle migration:

```
rails g migration AddStripeToProducts
```

et d'y ajouter la ligne suivante avant de passer la migration:

```
add_column :products, :stripe_price, :string
```


> __IMPORTANT__: Si vous avez déjà créé des produits, il est tout à fait possible que votre application et Stripe tirent la gueule car ces derniers n'auront pas leur "stripe_price". Dans mon cas, j'ai eu besoin de relancer un seed tout propre, en m'assurant qu'au moment du seed, chaque produit avait bien récupéré son "prix" Stripe.
>
> Et attention si vous utilisez un seed: n'oubliez pas de le modifier en conséquence!  

Voilà, maintenant que nous avons fait le lien entre notre BDD et les ID API des produits Stripe, passons à nos utilisateurs!

-------


### B. Lier nos utilisateurs à Stripe via le customer_id

Cette étape est primordial car c'est elle qui nous permettra par la suite d'offrir à nos utilisateurs la possibilité de modifier ou supprimer un abonnement. Le premier point ne concernera que les personnes n'ayant pas encore créé leur model utilisateur

**a. Je n'ai pas encore créé ma BDD**

Commençons par créer le model:

```
rails g model Users
```

puis dans la table de migration (dans `db/migrate`), il faudra ajouter ces lignes (en plus de celles liées à votre projet, bien sûr!):

```
t.string :plan
t.string :stripe_customer_id
t.string :session_token, :string
t.index :stripe_customer_id, unique: true
```
Dans notre cas, nous n'allons jouer qu'avec le stripe_customer_id, que nous allons créer juste après, mais ces colonnes permettront à Stripe d'identifier chaque utilisateur, ses différents achats, ses abonnements, etc. 

Vous pouvez maitenant passer la migration. 

**b. Créations d'une Customer stripe et attribution de son ID pour chaque nouvel utilisateur**

Rendez-vous dans le model User, où nous allons ajouter la méthode suivante :

```
  def create_stripe_customer
    customer = Stripe::Customer.create({
      email: self.email,
    })
    self.stripe_customer_id = customer.id
    self.save
  end
```

Elle va nous permettre de créer un nouveau client selon le modèle de Stripe qui une fois créé aura une id Stripe de type "`cus_unesuitedecaractères`", qui évidemment change pour chaque client. Il ne nous reste plus qu'à attribuer cette id à notre User fraichement créée, en l'ajoutant comme étant son "stripe_customer_id", puis de sauvegarder les changements. 

Pour finaliser cette étape, ajoutons ensuite en haut du model la ligne suivante pour utiliser le callback after_create

```
after_create :create_stripe_customer
```
Ainsi, chaque utilisateur créé aura automatiquement un Stripe ID attribué.

Parfait, maintenant que notre BDD est bien en place, passons aux choses sérieuses !


-----
## III. Permettre aux clients de payer un abonnement

Nous voilà arrivé à la parti un peu plus délicate de ce tuto. Pour que vous compreniez pourquoi, sachez que Stripe, au moment du paiement, établie une distinction claire entre un paiement unique et un abonnement, et que chacun a ses propres options, fonctionnalités, arguments, etc., qui ne correspondent évidemment pas à ceux de l'autre mode de paiement. Il y a donc deux possibilités :
  - vous prévoyez de ne proposer QUE des abonnements sur votre site et vous ne voulez QUE la solution pour intégrer le paiement de ses abonnements sur votre site : dans ce cas-là, rendez vous à la première sous-partie où je vous expliquerai comment faire
  - vous prévoyez de mettre en place à la fois des paiements uniques et à la fois des abonnements : dans ce cas-là, je vous invite à la lire la première sous-partie, afin de mieux comprendre ce que nous allons faire dans la seconde sous-partie, où je vous présenterai la méthode utilisée lors de mon projet final (que nous avons validé), pour permettre à ces deux méthodes de paiement de cohabiter.

### A. Mettre en place un unique système d'abonnements

**a. Le bouton de paiement**

Tout d'abord, nous allons créer le bouton qui renverra vers le système de paiement. Allez sur la page show de votre produit (et/ou ailleurs aussi, bien sûr) et ajoutez-y la ligne de code suivante : 
```
<%= button_to "Payer mon abonnement", checkout_create_path(price: product.stripe_price), class: "btn btn-primary", remote: true %>
```
Ce bouton va donc renvoyer vers la création d'un nouveau checkout (i.e. un écran de paiement) et nous lui passons en params le prix stripe du produit concerné (que vous aurez bien évidemment ajouté de votre côté!).

**b. Le controller Checkout et ses méthodes Create et Success**

Maintenant, tapez dans votre terminal :
```
rails generate controller checkout
```
Et rendez vous dans le controller en question (`app/controllers/checkout.rb`) pour que nous puissions y ajouter les méthodes nécessaires au bon fonctionnement de Stripe: Create et Success.

```
  def create
    @basket_price = params[:price].to_s
    session = Stripe::Checkout::Session.create(
      success_url: checkout_success_url + '?session_id={CHECKOUT_SESSION_ID}',
      cancel_url: checkout_cancel_url,
      payment_method_types: ['card'],
      customer: current_user.stripe_customer_id,
      mode: 'subscription',
      line_items: [{
        quantity: 1,
        price: @basket_price,
      }],
    )

    respond_to do |format|
      format.html { }
      format.js { }
    end
  end

  def success
    @session = Stripe::Checkout::Session.retrieve(params[:session_id])

    *mettez ici tout ce que vous voulez faire après un paiement réussi*
  end
```
Alors dans l'ordre, qu'avons nous fait ?
Dans la méthode create, tout d'abord:
  - Nous avons récupéré l'identifiant Stripe du produit (qui nous a été passé via les params du bouton de paiement, vous vous souvenez ?)
  - Nous demandons ensuite à Stripe de créer une nouvelle session de paiement à laquelle nous donnons tout plein d'arguments :
    * Les URLs de succès et échecs
    * La méthode de paiement (uniquement CB mais libre à vous de changer!)
    * L'ID Stripe du client en train de payer (si c'était vide, un nouvel ID serait créé - mais là, Stripe va pouvoir faire le lien directement avec notre current user et son ID Stripe !)
    * Le mode de paiement (c'est en quelques sortes ici que nous passons l'information la plus importante, puisque nous disons à Stripe : "ATTENTION, c'est un abonnement !)
    * Et enfin, les informations sur les produits: ici, la valeur doit rester à 1 puisque les utilisateurs ne prendront qu'un seul abonnement (mais cela peut être modifié évidemment) et le "price" est donc l'identifiant Stripe de notre produit, ce qui permettra d'afficher sur le portail de paiement les éléments indiqués lors de la création du produit sur le dashboard Stripe

Et enfin dans la méthode Success... et bin on y fait pas grand chose pour le moment, car tout ça dépend de ce que vous voulez y faire! Sachez simplement qu'une fois que le paiement sera passé, les params ne contiendront "que" l'ID de votre session de paiement. Vous pouvez donc faire appel à tous les arguments que vous voulez de cette session, donc n'hésitez pas à rajouter ce qui vous serait utile dans la méthode create. Pour plus d'informations sur les arguments d'une session de paiement : `https://stripe.com/docs/api/checkout/sessions/create`

Libre à vous ensuite de créer des mailers ou toute autre joyeuseté qu'un utilisateur peut attendre après un paiement.

Et il nous reste enfin une dernière étape...

**c. Changer la view Success**

Vous aurez peut-être remarqué que nous ne faisons pas appel au "PaymentIntent" qui est utilisé dans le tuto de Quentin.

Il s'agit en effet d'une option Stripe qui fonctionne en gros un peu comme un récapitulatif de paiement unique, mais qui n'est donc pas valable dans le cas d'un abonnement. Rendez-vous donc dans la view "Success" (`app/views/checkout/success.html.erb`) où vous pouvez mettre tout ce que vous voulez voir s'afficher sur la page de succès d'un paiement. 

### B. Combiner un système de paiement unique et d'abonnements

Les étapes sont les mêmes qu'au-dessus: créer des boutons personnalisés pour passer des infos via les params et ensuite adapter le controller Checkout.

**a. Les boutons de paiements**

*i. Bouton de paiement unique*
Voici le bouton à utiliser:
```
<%= button_to "Procéder au paiement", checkout_create_path(total: @cart.total_cart, payment_mode: "payment"), class: "btn btn-sprimary", remote: true %>
```
Comme vous pouvez le voir, nous avons fait passer deux params:
- le prix total à payer;
- et une variable créée arbitrairement, que j'ai ici appelée "payment_mode" à laquelle nous faisons passer une valeur en String : "payment". C'est la valeur de cet élément qui nous permettra de différencier au moment du checkout quel type de paiement est mis en place (nous reprenons les modes de paiement de Stripe pour plus de compréhension, mais ces valeurs sont indiquées arbitrairement et ne sont pas directement reliées à Stripe, j'aurais pu mettre n'importe quoi d'autre)

*ii. Bouton d'abonnement*

Voici le bouton à utiliser:
```
<%= button_to "Procéder à l'abonnement", checkout_create_path(price: product.stripe_price, payment_mode: "subscription"), class: "btn btn-primary", remote: true %>
```
Et nous avons donc dans les params:
- l'ID de produit Stripe (rien de surprenant)
- et une valeur différente pour "payment_mode"

**b. Le controller Checkout et ses méthodes Create et Success**

Cette partie est un peu plus longue que la précédente, et peut être plus compliquée à comprendre donc prenez bien le temps de la lire. Je vous explique juste après ce que nous y faisons.
Je préfère également signaler que cette construction du controller a été faite personnellement, pour notre projet final de la Session d'avril 2021 (projet validé, je vous rassure), et n'est donc pas directement issue d'un tuto Stripe officiel ou autre. 

```
def create
    @total = params[:total].to_d
    @basket_price = params[:price].to_s

    if params[:payment_mode] === "payment"
      @mode = "payment"
      @line_items = [
        {
          name: 'Rails Stripe Checkout',
          amount: (@total*100).to_i,
          currency: 'eur',
          quantity: 1
        },
      ]
    elsif params[:payment_mode] === "subscription"
      @mode = "subscription"
      @line_items = [
        {
          price: @basket_price,
          quantity: 1
        },
      ]
    end

    @session = Stripe::Checkout::Session.create(
      payment_method_types: ['card'],
      customer: current_user.stripe_customer_id,
      mode: @mode,
      line_items: @line_items,
      success_url: checkout_success_url + '?session_id={CHECKOUT_SESSION_ID}',
      cancel_url: checkout_cancel_url
    )

    respond_to do |format|
      format.html { }
      format.js { }
    end
  end

  def success
    @session = Stripe::Checkout::Session.retrieve(params[:session_id])
   

    if @session.mode === "payment"
      @payment_intent = Stripe::PaymentIntent.retrieve(@session.payment_intent)
       
    elsif @session.mode === "subscription"
      
    end

  end
```
Pour expliquer un peu plus clairement ce qu'il se passe ici:
- au début de notre méthode Create, nous récupérons les deux éléments qui vont nous permettre d'avoir un prix à payer : soit le total soit le prix du produit d'abonnement
- puis nous mettons ensuite en place la distinction entre les deux modes de paiement grâce à un if/elsif qui reprend les params et la valeur passée via "payment_mode", et qui va changer la valeur des deux arguments "mode" et "line_items" nécessaires à la création d'une session de paiement pour Stripe
- Enfin, nous appelons la création d'une nouvelle session de paiement via Stripe avec ces arguments

Puis dans la méthode success, nous faisons de nouveau appelle à la session de paiement Stripe, et créons de nouveaux une bifurcation grâce à un if/elsif pour obtenir une méthode Success adaptée au mode de paiement. 

**c. Changer la view Success**

Ici, rien de bien compliqué, il faudra simplement faire de nouveau appel à notre if/elsif, mais sur une page html.erb. Il vous faudra ensuite personnaliser les éléments que vous voulez mettre dans chacune des deux options, je ne vous ai mis ici que des exemples sommaires:

```
<% if @session.mode === "subscription" %>
  <p>Nous avons bien reçu votre inscription à notre abonnement ! Merci :)</p>
<% elsif @session.mode === "payment" %>
  <p>Nous avons bien reçu votre paiement de <%= number_to_currency(@payment_intent.amount_received / 100.0, unit: "€", separator: ",", delimiter: "", format: "%n %u") %>.</p>
  <p>Le statut de votre paiement est : <%= @payment_intent.status %>.</p>
<% end %>
<%=link_to "Retour à l'accueil", root_path, class:"btn btn-dark"%>
```

-----------------

Récapitulons. Jusqu'à présent nous avons:
- créé des produits d'abonnement sur le dashboard Stripe
- relié ces produits avec notre BDD
- permis à notre app de faire le lien entre nos utilisateurs et la fonction "Customer" de Stripe, pour s'assurer que chaque utilisateur garde bien la trace de tous ses paiements
- installé le paiement des abonnements sur notre application

Il ne nous manque plus qu'une seule étape vous ne croyez pas ?

Eh oui, à aucun moment nous ne permettons à nos utilisateurs de modifier leur abonnement ! Pour cela, passez à la dernière partie ci-dessous :)



-----------------

## IV. Créer le portail Client de Stripe


Ce portail Client est une page externe à votre app, mise en place par Stripe, qui permettra à chaque client de gérer les abonnements auxquels il aura souscris: il pourra les suspendre, les annuler ou même changer d'option si vous l'avez mis en place. Plutôt pratique pour nos utilisateurs donc !

On va commencer par créer un nouveau controller :

```
rails generate controller CustomerPortalSessions
```

puis rendez-vous y et ajoutez les lignes suivantes : 

```
  def create
    portal_session = Stripe::BillingPortal::Session.create(
      customer: current_user.stripe_customer_id,
      return_url: "https://l'adresse-de-votre-choix/"
    )
    redirect_to portal_session.url
  end
```

Comme vous pouvez le voir, c'est donc ici que nous faisons appel à notre stripe_customer_id pour que le portail créé soit bien relié à notre utilisateur actuellement connecté, pour être sûr qu'il ne récupère pas les informations du client d'avant ou d'après. Car vous verrez notamment que ce portail offre aussi la possibilité d'enregistrer ses informations de paiement, ses informations personnelles, etc. On va donc éviter que votre client numéro 12 récupère le code de CB de votre client numéro 11, et inversement!
Vous pouvez aussi choisir un url qui sera rattaché au bouton "Retour" de ce portail client.

Maintenez rendez-vous dans votre fichier routes.rb et ajoutez-y la ligne suivante :

```
resources :customer_portal_sessions, only: [:create] 
```

Car en effet, vous n'aurez (normalement) plus jamais besoin de toucher à ce controller et surtout d'y ajouter de nouvelles méthodes. Donc gardons uniquement la route pour la méthode create (ce n'est pas obligatoire, mais ça fait plus propre!).

Et enfin, dernière étape, maintenant que nous avons une route, il va bien falloir l'appeler quelque part! Rendez-vous donc cette fois-ci dans vos views et ajoutez, sur la page que vous voulez (probablement une page de profil) le lien suivant :

``` 
<%= link_to "Gérer mes abonnements", customer_portal_sessions_path, method: "POST" %> 
```

Et TADAM, vos utilisateurs auront maintenant accès à leur propre portail de gestion de leurs abonnements, mais aussi de leurs informations de paiement!

Maintenant, pour avoir un portail client encore plus stylé, je vous invite à retourner sur votre dashboard Stripe, et aller dans `Paramètres > Billing > Portail Client`. Vous pourrez ici le personnaliser, en termes de contenu (mettre les liens vers vos politiques internes, etc.) mais aussi pour y ajouter ou enlever certaines fonctionnalités pour vos futurs clients (mettre à jour leurs moyens de paiements, leurs infos personnels, etc.). 

N'hésitez pas non plus à explorer le dashboard Stripe, vous y découvrirez tout un tas de fonctionnalités!