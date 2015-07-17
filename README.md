# Kerberos and Hadoop: The Madness beyond the Gate


> The most merciful thing in the world, I think, is the inability of the human mind to correlate all its contents.
> We live on a placid island of ignorance in the midst of black seas of infinity, and it was not meant that we should voyage far.
> The sciences, each straining in its own direction, have hitherto harmed us little;
> but some day the piecing together of dissociated knowledge will open up such terrifying vistas of reality,
> and of our frightful position therein, that we shall either go mad from the revelation
> or flee from the light into the peace and safety of a new dark age.

> *[The Call of Cthulhu](https://en.wikisource.org/wiki/The_Call_of_Cthulhu), HP Lovecraft, 1926.*

## Contents

* [The Madness beyond the gate](sections/kerberos_the_madness.md)
* [What is Kerberos?](sections/what_is_kerberos.md)
* [Hadoop and Kerberos](sections/hadoop_and_kerberos.md)
* [HDFS and Kerberos](sections/hdfs.md)
* [UGI](sections/ugi.md)
* [Testing](sections/testing.md)
* [Low-Level Secrets](sections/secrets.md)
* [Error Messages to Fear](sections/errors.md)
* [Checklists](sections/checklists.md)
* [Glossary](sections/glossary.md)
* [Bibliography](sections/biblography.md)
* [Acknowledgements](sections/acknowledgements.md)

## Disclaimer

Just as the infamous [Necronomicon](http://www.amazon.com/gp/product/0380751925) is a collection
of notes scrawled in blood as a warning to others, this book is

1. Incomplete.
1. Based on experience and superstition, rather than knowledge and insight.
1. Covers knowledge that will drive the reader insane.

Reading this book implies recognition of these facts and that the reader, their estate and
their heirs accept all risk and liability. The author is not responsible if anything happens
to the Hadoop clsuter, including all the data stored in HDFS disappearing into an unknown dimension,
or the YARN scheduler starting to execute jobs to summon hitherto unknown pre-human deities.

**You have been warned**


## Implementation notes.

This is a work in progress book designed to built using the [gitbook tool chain](https://github.com/GitbookIO/gitbook),
the best OSS implementation to date of the book-as-software process proposed in
[Refactoring the Publishing Process](http://people.apache.org/~stevel/papers/refactoring_publishing.pdf),
Loughran and Hatcher, 2002.

It is hosted on [github](https://github.com/steveloughran/kerberos_and_hadoop)
