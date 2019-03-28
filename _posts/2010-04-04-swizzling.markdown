---
layout: post
title:  "Swizzling de méthode et Refléxivité par l'exemple"
date:   2010-04-04 19:45:38 +0800
---

Objective-C est un language très dynamique. Cela signifit par exemple qu’il peut résoudre des appels a méthode au runtime (lors de l’execution de programme, et non à la compilation), mais aussi effectuer de la refléxivité sur ses objets (Au runtime, toujours, accéder à des informations sur un objet, comme ses attributs et méthodes) ou même les modifier !

Pour s’amuser un peu avec ces fonctionnalités avançées d’Objective C, nous allons prendre un exemple simple. La methode `description` qui appartient à NSOBject, permet d’afficher des informations sur l’objet. Par défaut elle n’affiche que son nom et son adresse, mais un object peut implémenter une surcharge, puisque tout objet hérite de NSObject :

{% highlight bash %}
> [myFoobar description]
<FoobBar:0x3e16820>
> [myNSArry description]
(
    lol,
    42
)
{% endhighlight %}

Nous allons développer une meilleure méthode `description`, qui liste les attributs de l’objet qui l’appelle (on l’appelle le « sender »), et affiche son type, son nom et sa valeur. Comme Objective C est dynamique, nous allons echanger les implémentations au runtime : les appels à `description` iront à notre méthode, et nous décideront dans celle-ci si nous appellons la méthode `description` originale (si l’objet l’implémente) ou plutôt la nôtre.

Catégories et Swizzling
Pour rajouter des méthodes à une classe, Objective C utilise les catégories. Créons donc une catégorie à NSObject afin d’y ajouter notre méthode :

{% highlight objc %}
@interface NSObject (DTFObject)
 
@end
{% endhighlight %}

Le problème c’est que si nous implementons une methode `description` dans notre catégorie, la méthode `description` du système va être écrasée. En effet, les catégories ne fonctionnent pas comme la surcharge de méthode dans ce cas. Puisque nous voulons faire appel a la méthode du systeme lorsque l’objet l’implémente, nous allons faire du swizzling de méthode : lors de l’execution, les implémentation de notre methode `dtfDescription` et de description seront echangées. Exemple :

{% highlight objc %}
[foo description];        // execute [foo dtfDescription]
[foo dtfDescription];    // execute [foo description]
{% endhighlight %}

Nous utiliserons [JRSwizzle](https://github.com/rentzsch/jrswizzle/), une implémentation libre qui permet le swizzling de méthode dans n’importe quelle version d’Objective C. Nous faisons l’appel à `jr_swizzleMethod` dans la méthode `+ (void)load`. Cette méthode est invoquée lorsqu’une classe ou une catégorie (notre cas) est ajoutée au runtime :

{% highlight objc %}
static void SwizzleClassMethods(Class class, SEL firstSelector, SEL secondSelector);
 
@implementation NSObject (DTFObject)
 
+ (void)load
{
    NSError *err = [[NSError alloc] init];
 
    [self jr_swizzleMethod:@selector(description) withMethod:@selector(dtfDescription) error:&err];
    if ([[err userInfo] valueForKey:@"NSLocalizedDescriptionKey"] != nil)
        NSLog(@"%@", [[err userInfo] valueForKey:@"NSLocalizedDescriptionKey"]);
    [err release];
}
 
@end
{% endhighlight %}

Maintenant que les méthodes sont échangées il faut implémenter la nôtre. Nous souhaitons tout d’abord savoir si le sender (pour rappel c’est l’objet qui appelle notre méthode) implémente description. Objective C propose de savoir cela via `respondsToSelector:`, mais comme il retourne vrai si la méthode est implémentée dans un parent, elle retournera vrai dans tout les cas ici (NSObject est le parent de tout objet). Pas de problème, utilisons la reflexivité pour savoir cela. Nous allons récupérer la liste des méthodes du sender via `class_copyMethodList()` puis comparer son nom avec le selector qui nous interresse (`description`) via `sel_isEqual()` et `method_getName()` :

{% highlight objc %}
- (BOOL)implementsSelector:(SEL)aSelector
{
    Method          *methods;
    unsigned int    count;
    unsigned int    i;
 
    methods = class_copyMethodList([self class], &count);
    for (i = 0; i < count; i++)
        if (sel_isEqual(method_getName(methods[i]), aSelector))
            return YES;
    return NO;
}
{% endhighlight %}

Implémentons maintenant cela dans notre méthode :

{% highlight objc %}
- (NSString *)dtfDescription
{
    if ([self implementsSelector:@selector(description)])
        return [self dtfDescription];
}
{% endhighlight %}

Attention : nous ne créeons pas une boucle récursive infinie en retournant `[self dtfDescription]` ! Souvenez vous que les appels sont interchangés. La méthode appellée sera `description`. Bouclons maintenant sur les attributs de notre objet :

{% highlight objc %}
- (NSString *)dtfDescription
{
    unsigned int    i;
    unsigned int    count;
    Ivar            *ivars;
    NSString        *type;
    NSString        *key;
    NSMutableString *description;
 
    if ([self implementsSelector:@selector(description)])
        return [self dtfDescription];
    ivars = class_copyIvarList([self class], &count);
    description = [NSMutableString stringWithFormat:@"<%s:0x%06x>\n(\n", class_getName([self class]), self];
    for (i = 0; i < count; i++)
    {
        type = [[NSString alloc] initWithUTF8String:ivar_getTypeEncoding(ivars[i])];
        key = [[NSString alloc] initWithUTF8String:ivar_getName(ivars[i])];
    }
    [description appendString:@")"];
    return description;
}
{% endhighlight %}

Nous en profitons pour commencer a remplir notre string de retour comme le faisais l’ancien `description` : avec le type et l’adresse de l’objet. Maintenant récuperons chaque attribut, et si c’est un attribut simple (char, char *, int…) affichons sa valeur, sinon son nom, type et adresse. Et nous avons fini. Voici la méthode finale :

 
{% highlight objc %}
- (NSString *)dtfDescription
{
    unsigned int    i;
    unsigned int    count;
    Ivar            *ivars;
    NSString        *type;
    NSString        *key;
    id              object;
    NSMutableString *description;
 
    if ([self implementsSelector:@selector(description)])
        return [self dtfDescription];
    ivars = class_copyIvarList([self class], &count);
    description = [NSMutableString stringWithFormat:@"<%s:0x%06x>\n(\n", class_getName([self class]), self];
    for (i = 0; i < count; i++)
    {
        type = [[NSString alloc] initWithUTF8String:ivar_getTypeEncoding(ivars[i])];
        key = [[NSString alloc] initWithUTF8String:ivar_getName(ivars[i])];
        if ((object = object_getIvar(self, ivars[i])) != nil)
        {
            if ([type characterAtIndex:0] == _C_ID)
                [description appendString:[NSString stringWithFormat:@"\t%@ %@ = 0x%6x\n", type, key, object]];
            else
                [description appendFormat:[NSString stringWithFormat:@"\t%@ = %%%@\n", key, type], object];
        }
        else
            [description appendFormat:@"\t%@ = nil 0x%06x", key, object];
        [type release];
        [key release];
    }
    [description appendString:@")"];
    return description;
}
{% endhighlight %}

Le code complet est disponible sur github : [https://github.com/thomasjoulin/DTFObject]()