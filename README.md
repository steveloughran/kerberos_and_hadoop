
# Hadoop and Kerberos: The Madness beyond the Gate


> The most merciful thing in the world, I think, is the inability of the human mind to correlate all its contents.
> We live on a placid island of ignorance in the midst of black seas of infinity, and it was not meant that we should voyage far.
> The sciences, each straining in its own direction, have hitherto harmed us little;
> but some day the piecing together of dissociated knowledge will open up such terrifying vistas of reality,
> and of our frightful position therein, that we shall either go mad from the revelation
> or flee from the light into the peace and safety of a new dark age.

> *[The Call of Cthulhu](https://en.wikisource.org/wiki/The_Call_of_Cthulhu), HP Lovecraft, 1926.*


This manuscript discusses low-level issues related to Apache&trade; Hadoop&reg; and Kerberos

## Disclaimer

Just as the infamous [Necronomicon](http://www.amazon.com/gp/product/0380751925) is a collection
of notes scrawled in blood as a warning to others, this book is

1. Incomplete.
1. Based on experience and superstition, rather than understanding and insight.
1. Contains information that will drive the reader insane.

Reading this book implies recognition of these facts and that the reader, their estate and
their heirs accept all risk and liability. The author is not responsible if anything happens
to their Apache Hadoop cluster, including all the data stored in HDFS disappearing into an unknown dimension,
or the YARN scheduler starting to summon pre-human deities.

**You have been warned**


## Implementation notes.

1. This is a work in progress book designed to built using the [gitbook tool chain](https://github.com/GitbookIO/gitbook).

1. It is hosted on [github](https://github.com/steveloughran/kerberos_and_hadoop).
Pull requests are welcome.

1. All the content is Apache licensed.

1. This is not a formal support channel for Hadoop + Kerberos problems. If you have a support
contract with [Hortonworks](http://hortonworks.com/) then issues related to Kerberos may 
eventually reach the author. Otherwise: try 

      - [Hortonworks Answerhub](https://community.hortonworks.com/answers/index.html)
      - The users mailing list of Apache Hadoop, the application and you are using on top of it.
      - [Stack Overflow](http://stackoverflow.com/search?q=hadoop+kerberos).

