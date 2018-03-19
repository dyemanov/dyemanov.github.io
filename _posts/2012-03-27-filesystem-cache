---
layout: post
title: Firebird vs Windows: file-system caching issue
---

This article is going to shed some light on the facts related to the issue registered as [CORE-3791](http://tracker.firebirdsql.org/browse/CORE-3791) in the project bug tracker. Let's start with some links.

The origin of the problem is described here:
http://support.microsoft.com/kb/2549369

A lot of additional useful information can be found here (especially in the comments):
http://blogs.msdn.com/b/ntdebugging/archive/2007/11/27/too-much-cache.aspx

In fact, this issue consists of two slightly different but interrelated parts.

The first one is that the file-system cache size is unlimited by default in Windows 64-bit, so its working set can consume all the available RAM, thus forcing almost everything else to swap and finally starting to swap itself. It's visually detected by a slow system response and lots of the HDD activity. The suggested solution is to limit the file-system cache size with the [SetSystemFileCacheSize](http://msdn.microsoft.com/en-us/library/aa965240.aspx) API function. And Firebird 2.5 supports this solution with the *FileSystemCacheSize* setting in firebird.conf.

Unfortunately, we have reports that the problem may still take place for some customers. I have performed a research and the findings are the following:

1. In order to change the file-system cache size, the user running the application must have the priviledge named *SeIncreaseQuotaPrivilege*. This fact is documented in firebird.conf. By default, only LocalSystem and the Administrators group have this priviledge assigned. The Firebird service installation utility instsvc grants this priviledge to the custom user account chosen during the installation. So far so good.
2. However, this may not work if you run Firebird as an application or use it in the embedded mode. In this case and with the default UAC settings, Firebird will not have the sufficient permissions even if the currently logged user is an administrator or has *SeIncreaseQuotaPrivilege* explicitly assigned to the current user in the Local Security Policies. In order to make it working, you need to start Firebird (or the host application using fbembed.dll) using the UAC's feature "Run as Administrator".
3. The worst thing here is that it could be problematic to figure out whether the file-system cache size has been actually limited or not. Contrary to what the comment inside firebird.conf says, the lack-of-permission error is unlikely to be printed to firebird.log and it makes the diagnostics hardly possible.
4. And finally, 32-bit builds of Firebird may completely ignore the *FileSystemCacheSize* configuration setting if they run on a 64-bit Windows box. This is because the default cache size limit is too high to be represented with the 32-bit Windows API.

The second part of the issue is that even if your setup handles the file-system cache size limit properly and thus avoids any possible out-of-memory conditions, you may still notice the active swapping and the related performance issues. The explanation is simple - the aforementioned approach doesn't limit the cache per se, it just limits its working set, i.e. pages kept resident in RAM.

Now we get closer to the root of the whole issue. If the application requests random file access, and this is what Firebird does, it forces the Windows cache manager to "pin" all the visited pages in a hope to have them accessed again. In other words, those pages are never removed from the cache. With the working set size being limited, the cache manager has no other choice but to store the non-resident part (pages that don't fit the working set) in the pagefile. I know it sounds completely crazy and I could never imagine that caching of a disk access might be implemented via yet another disk access. But it appears to be true. You can see the similar conclusions published for another affected application (IBM Lotus Domino):
http://blog.nashcom.de/nashcomblog.nsf/dx/new-performance-problem-with-domino-on-windows-2008-64bit.htm

So the only effective solution seems to disable the random access request (i.e. remove the FILE_FLAG_RANDOM_ACCESS flag) from the Windows API calls used to create/open the files. Moreover, in this case the file-system cache size limit should not be actual anymore, as Windows won't be expanding the cache out of the reasonable boundaries. The quick tests prove this solution being workable. Below are the results observed for a 8GB database on a box with 4GB RAM (both Firebird and Windows are 64-bit):

- More or less sequential access (natural table scan): ~ 2m00s vs 1m10s
- Mixed sequential and random access (natural table scan + external sorting): ~ 3m20s vs 1m30s
- Random access (table scan via non-PK index based ordering): ~ 15m vs 12m

So we can expect up to 2x performance improvement but mostly for cases when the whole database or its major part has to be accessed (e.g. during backup or after its completion).

I've also attempted to find out how other open source databases work with files on Windows and here are the results:

- MySQL / InnoDB - random access is not requested
- PostgreSQL - random access is not requested
- SQLite - random access was requested, but disabled in 2008

Here is the quote of the commit log for SQLite:

> Iterating through all the features of a large SDF file causes a significant loss of the system available memory. This is due to the system disk cache which tend to use all the memory it can get. The fix is to remove the random access option when opening the SDF file. That reduced the memory used by the disk cache to a reasonable amount and did not impact the performance of the SDF provider (it actually helped a bit).

As we have seen above, it really helped for Firebird as well. But so far we were speaking about large databases with sizes exceeding the available memory amounts. So the only remaining question is how this change would affect small databases that can fit into the file-system cache completely. I've repeated the aforementioned test with a 1GB database and results are the following:

- More or less sequential access (natural table scan): ~ 4.2s vs 4.5s
- Mixed sequential and random access (natural table scan + external sorting): ~ 12.4s vs 12.7s
- Random access (table scan via non-PK index based ordering): ~ 1m10s vs 1m30s

So we see a very small penalty for sequential or mixed access but a noticeable penalty (about 30%) for highly random access. This means that even if there's enough RAM to cache the whole database, the Windows cache manager prefers to work with a small working set unless the random access is explicitly requested. As for me, this is also quite unexpected from a decent cache manager, but perhaps it depends on the various Windows settings (*LargeSystemCache* etc).

Please beware that the worst possible result is achieved with a test pattern rarely used in production. Also, it can be compensated by a larger Firebird page cache (remember that databases in question are small enough and hence you have plenty of the unused RAM). Taking this into account, as well as the experience of other databases, this solution has been committed into Firebird 2.1.5, Firebird 2.5.2 and Firebird 3.0 branches.

*P.S. One might think that Firebird could be smart enough and decide about the random access mode dynamically depending on the database size vs the RAM size or even better vs the current file-system cache limit. But it doesn't sound as a good idea. There may be many databases accessed on the same server, databases may grow while being connected and so on. It's practically impossible to handle all these cases properly.*
