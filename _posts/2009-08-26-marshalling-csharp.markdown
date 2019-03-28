---
layout: post
title:  "Manipuler les buffers d'une librairie non-managée en C#"
date:   2009-08-26 19:45:38 +0800
---

En C#, un langage managé, utiliser une dll qui prends un buffer en paramètre n’est pas évident au premier abord : il n’y a pas de notion de pointeur dans un langage managé puisque le garbage collector est seul maitre de la mémoire. Ainsi, si votre dll prends un pointeur sur un tableau de caractère (un buffer alloué qu’il va remplir et que vous souhaitez par la suite manipuler), vous devez trouver une solution pour faire passer votre tableau C# pour un pointeur sur un tableau de char. Plusieurs solutions existent, celle-ci a le mérite de ne pas à utiliser la clause `unsafe`.

Il s’agit de [marshalling][marshalling]. ou sérialisation binaire. On prends un tableau de `byte` csharp, et on le transmet comme un pointeur sur un tableau non managé. Le code est plutôt simple d’accès, quelques explication en dessous :

{% highlight csharp %}
using System;
using System.Runtime.InteropServices;
 
namespace Test
{
    class Program
    {
        public const Int32 BUFF_SIZE = 1024;
 
        [DllImport("library.dll")]
        static extern Int32 foo([MarshalAs(UnmanagedType.LPArray)] byte[] buffer);
 
        static void Main(string[] args)
        {
            Int32   len;
            byte[]  buffer = new byte[BUFF_SIZE];
            int     i;
            string  str;
 
            len = foo(buffer);
            for (i = 0; i < len; i++)
            {
                str += Convert.ToChar(buffer[i]);
            }
        }
    }
}
{% endhighlight %}

Concrètement, la ligne qui nous interresse est :

{% highlight csharp %}
        static extern Int32 foo([MarshalAs(UnmanagedType.LPArray)] byte[] buffer);
{% endhighlight %}

Elle signifit que vous allez sérialiser votre tableau de `byte` en pointeur sur un tableau non-managé. Le prototype de votre methode `foo()` dans la dll serait ici :

{% highlight csharp %}
        long foo(char *buffer);
{% endhighlight %}

[marshalling]: https://fr.wikipedia.org/wiki/Marshalling
