---
layout: post
title:  "API http://is.gd : Introduction à l’intégration de Web Services sous iOS"
date:   2010-10-27 19:45:38 +0800
---

La plupart des applications se connectent à une base de donnée pour afficher leurs données. Sur mobile notamment, il arrive souvent que cette source de donnée soient stockée sur un serveur externe, accessible via des requêtes à un API, le plus souvent via HTTP. On appelle cela un Web Service.

Ce tutorial est destiné aux débutant sous iOS. Nous prendrons l’exemple concret de la communication avec une API simple, sans authentification : http://is.gd. L’objectif est simple. Envoyer une requête à http://is.gd en fournissant en paramètre une url longue, et récupérer la version courte, le tout sans latence (requête asynchrones).

Etablissons tout d’abord notre classe `URLShortenerController`. Nous voulons un `NSMutableData` pour stocker tout retour éventuel du Web Service, et nous avons besoin d’une méthode `initWithDelegate:url:` qui va initialiser notre instance en stockant l’instance qui souhaite avoir le retour (delegate), et l’url a raccourcir :

{% highlight objc %}
@protocol URLShortenerDelegate;
 
@interface URLShortenerController : NSObject
{
    id delegate;
@private
    NSMutableData *data;
}
 
@property (nonatomic, assign) id delegate;
 
- (id)initWithDelegate:(id)aDelegate url:(NSString *)anURL;
 
@end
{% endhighlight %}

Nous voulons donc envoyer aussi un retour quand il est disponible. Nous allons donc créer un protocole, que toute les classes qui souhaitent utiliser notre raccourcisseur d’url doivent implémenter.

{% highlight objc %}
@protocol URLShortenerDelegate
@optional
- (void)shortener:(URLShortenerController *)shortener didSucceedWithShortenedURL:(NSString *)url;
- (void)shortener:(URLShortenerController *)shortener didFailWithError:(NSError *)error;
@end
{% endhighlight %}

Dans notre fichier d’implémentation maintenant, nous initialisons l’instance comme prévue, et nous créeons une instance de `NSURLConnection`. Cette classe permet d’entreprendre et récupérer de manière asynchrone des données :

{% highlight objc %}
- (id)initWithDelegate:(id<URLShortenerDelegate>)aDelegate url:(NSString *)anURL
{
    if ([super init])
    {
        data = [[NSMutableData alloc] init];
        self.delegate = aDelegate;
        NSURL *requestURL = [NSURL URLWithString:[NSString stringWithFormat:@"http://is.gd/api.php?longurl=%@", anURL]];
        NSURLRequest *request = [NSURLRequest requestWithURL:requestURL];
        NSURLConnection *connection = [NSURLConnection connectionWithRequest:request delegate:self];
    }
 
    return self;
}
{% endhighlight %}

Nous pouvons implémenter les callbacks de `NSURLConnection`. Je ne vous montrerais que les deux première, les autres étant utile en cas d’erreur. Tout d’abord lorsque nous recevons des données, nous les ajoutons à notre `NSMutableData`

{% highlight objc %}
- (void)connection:(NSURLConnection *)connection didReceiveData:(NSData *)someData
{
    [data appendData:someData];
}
{% endhighlight %}

Ensuite quand nous avons reçu toute les données, nous pouvons appeller notre delegate pour lui fournir l’url raccourcie :

{% highlight objc %}
- (void)connectionDidFinishLoading:(NSURLConnection *)connection
{
    NSString *url = [[[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding] autorelease];
    [delegate shortener:self didSucceedWithShortenedURL:url];
}
{% endhighlight %}

Et voila ! Un peu d’imagination et vous pourrez intégrer un vrai Web Service. Les bases fournies ici seront les même.