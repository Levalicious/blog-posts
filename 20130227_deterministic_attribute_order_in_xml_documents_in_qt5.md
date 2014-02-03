In [Enigma Studio](http://www.braincontrol.org/enigma.php) we use Qt's XML framework to save and load projects. We use text-based XML files in conjunction with the merge/diff tools of our version control system to collaboratively work on the same project. With little additional internal organization (who works on what) this gives a simple but powerful workflow without writing any additional tool code.

Recently, after switching from Qt 4.8.1 to Qt 5.0.0 we observed that merging projects was not longer possible. The reason for this was that the order of XML attributes changed between successive savings of the very same project, even the project remained completely unchanged. According to the XML standard this is fine as XML is order independent. However, I decided to report the observed behavior as a bug, as I thought that many people require XML files saved by Qt's XML framework to be mergeable/diffable. Unfortunately, the Qt team was of a different opinion. They do not want to enforce a particular attribute ordering to have more freedom for future code optimizations and hence, recommend using an additional library (e.g. [libxml2](http://www.xmlsoft.org/) to convert XML documents to [canonical XML](http://en.wikipedia.org/wiki/Canonical_XML) in a post-processing step. As that approach would add an additional library  dependency we started to investigate our-selves.

It turned out that [salted](http://en.wikipedia.org/wiki/Salt_\(cryptography\)) hash functions were the root of the problem. `QHash`, which is heavily used by the XML framework, salts Qt's hash functions with a random seed which is determined on first usage of `QHash`. The salting is applied to make certain security attacks impossible (e.g. pre-calculating a set of keys with equal hashes, effectively decreasing the runtime complexity for insertion of `QHash` from amortized `O(1)` to `O(n)` because of collisions. For more information look [here](http://qt-project.org/doc/qt-5.0/qtcore/qhash.html#algorithmic-complexity-attacks). Fortunately, it turned out that the applied salt can be specified by defining an environment variable called `QT_HASH_SEED`. That environment variable can be either specified system/user-wide, or process-wide on application start-up by calling `qputenv("QT_HASH_SEED", "12345")`.

Note, that this line of code has to be called before any Qt code which uses `QHash` is executed, because the environment variable is for performance reasons only read once. Especially, it has to be called before instancing an `QApplication` object. Using `qputenv` only changes the process' environment block. When child processes are spawned they inherit by default the environment block of their parent process and thus, do not require that the environment variable is defined for them again.