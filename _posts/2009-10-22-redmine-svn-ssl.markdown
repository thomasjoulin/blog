---
layout: post
title:  "Configurer Redmine pour un dépot Subversion sous SSL"
date:   2009-10-22 19:45:38 +0800
---

Si l’onglet dépot de votre Redmine vous affiche ce message d’erreur : The entry or revision was not found in the repository, c’est que Redmine n’arrive pas à se connecter à votre dépot Subversion. Si vous arrivez à checkout en utilisant les même identifiants/passe que ceux fournis à redmine, et que votre dépot Subversion est accessible via SSL (adresse du dépot en HTTPS), votre problème vient peut être du fait que Redmine ne peut accepter seul les certificats SSL.

### Accepter les certificats SSL
Connectez-vous sur votre serveur via ssh, et authentifiez vous avec les identifiants/passe fournis a redmine. Puis, à la racine du compte, faites un checkout de votre dépot et acceptez definitivement le certificat.

{% highlight bash %}
$> cd
$> svn co https://svn.your-server.tld
Error validating server certificate for 'https://svn.your-server.tld:443':
 - The certificate is not issued by a trusted authority. Use the
   fingerprint to validate the certificate manually!
Certificate information:
 - Hostname: svn.your-server.tld
 - Valid: from Thu, 01 Jan 1970 00:00:00 GMT until Sun, 01 Jan 2042 00:00:00 GMT
 - Issuer: Your-Company
 - Fingerprint: 4a:db:a5:a3:b3:48:9d:d2:f4:f9:66:0f:49:c4:84:2e:cc:55:83:7c
(R)eject, accept (t)emporarily or accept (p)ermanently? p
$> /etc/init.d/mongrel_cluster restart
{% endhighlight %}

Testez, vous devriez normalement avoir accès à votre dépot. Si ce n’est pas le cas, c’est que Redmine ne cherche pas la configuration de votre subversion au bon endroit. La suite de cet article résoud ce problème.

### Configurer Redmine pour la gestion du dépot SVN
Typiquement, votre dossier d’installation de redmine appartient à l’utilisateur redmine et au groupe www-data. Redmine cherche donc la configuration de subversion dans ~redmine/.subversion, et c’est pour cela que nous avons fait un checkout dans ce dossier (accepter le certificat l’ajoute dans le dossier .subversion de l’utilisteur). Apparement redmine ne cherche pas dans ce dossier. Ouvrer le fichier du configuration de svn dans redmine, et editez la commande utilisée pour appeller le binaire en lui passant en paramètre le bon dossier de configuration.

{% highlight bash %}
$> emacs dir-to-redmine/lib/redmine/scm/adapters/subversion_adapter.rb
{% endhighlight %}

Remplacez SVN_BIN = "svn" par SVN_BIN = "svn --config-dir /home/redmine/.subversion"

Redemarrez mongrel_cluster, et voilà !