# Reverse-engineering [Book Guard Pro](http://bookguardpro.com/home.php)

I came across this website (home page saved to `./http:/bookguardpro.com/home.php`; SHA-256 = a797177fa63b0c28a99442d77eb6c6c5452f440c4c3469a4737c7895dec1dde1) offering a DRM service, and decided to reverse-engineer it. The site doesn't look very professional; not expecting much difficulty. The site offers for download a sample protected file (http://bookguardpro.com/TryBookGuardPro1.exe, also saved; SHA-256 = 1c414bd221c8feddddaef9ca9c0e1889067c22d57110ed9928ba686de2ed7967). It looks like an ordinary static web server; nothing special here. The home page provides the password (K5PHQCXS) that will supposedly be needed.

## The relevant claims

> (Worry no longer!  See what you get; download FREE LIVE EXAMPLE below on this page )

(below)

> ## Experience a FREE Live Sample of How BookGuard Pro's PDF Security Works...
>
> Try this FREE sample protected PDF File below (zero-login and 100% safe)
>
> CLICK HERE<br/> to download and test for FREE this protected PDF file I secured earlier today with BookGuardPro (simply save this file to a folder on your desktop).
>
> Next, double-click the .exe file you just downloaded. When asked by the program, enter the temporary Licence Code below, I'm giving you so you can see a very important, secret message in the PDF (I'm not joking ...you must know this insider info about PDFs ...if your business sells or deals in them in anyway.
>
> Your limited-time Licence Code is;
>
> K5PHQCXS
>
> After download, you will see the BookGuardPro file open up when you click it. But there is no way you can save, edit or print the PDF inside.
>
> What's more, even though we are thousands of miles away from each other, I could remove your viewing access to this PDF in just seconds ...anytime I need to!

# What I tried

`file` identifies it as a PE32 executable. I tried extracting resources from the exe using `wrestool`; nothing useful.

Then I tried running it under Wine (`kdesudo -u altuser ./TryBookGuardPro1.exe`). It popped up a dialog asking me for the password. After I entered the password, it displayed a message saying it was verifying the license (presumably communicating with the license server). (While it would possible to obfuscate this network traffic using public-key cryptography and an RNG in the client, judging by the quality of the rest of the DRM I suspect they didn't even bother (which would make it vulnerable to a replay attack). Haven't tried sniffing network traffic.)

It then popped up a PDF viewer displaying the protected document. (The first page displays the error "Couldn't render the page", but pages 2-3 have some advertising copy.) Looking in the Help menu, it turns out that this viewer is a presumably modified version of [Sumatra PDF](http://code.google.com/p/sumatrapdf/). Selecting text and file operations are disabled.

Next I tried to see whether it stores any temporary files (using `sudo -u altuser lsof +L5`). I found it created the temp directory `~altuser/.wine/drive_c/users/altuser/Temp/sysPro_bgf6.tmp` (`bgf6` is randomized on each invocation). Though Sumatra doesn't keep the file open, the program does keep open this temp directory. Browsing there in the file browser reveals that alongside the temp directory there is a small file `BGPexample.pdf` (SHA-256 = 6c4f2d79a12e87c5b9cb61bef2c79fa9f727db6c99efa25dd677ee067f3e0c67) containing only a message saying that the product is protected with BGP. This file is presumably there for misdirection.

Within the temp directory there is a larger `.pdf` file with the same filename (SHA-256 = 47075032cfd3eb0cf5d2ee61c80e4d72c868b90378b6dd149461d622b75afe10), and a `SumatraPDF.exe` executable. However, it's not a valid PDF. In a hex editor, it looks like a PDF file except with the PDF header replaced with `%BGP-1.5.%`. Unfortunately changing this to `%PDF-1.5.%` doesn't produce an openable PDF.

Running `SumatraPDF.exe BGPexample.pdf` launches the same viewer displaying the "protected" document, with no delay (apparently no network access)! This means that anyone can completely bypass the protection after running it once, by copying both the obfuscated PDF and the executable. Oops. Reverse-engineering this `SumatraPDF.exe` would let you convert to other formats (haven't tried this). Incidentally, closing the viewer launched by the main application deletes the misdirection file, but not the temp directory (leaving the unprotected file on the filesystem). Oops.

When running the launcher, in the console it prints a Wine message `fixme:cacls:main This is dummy cacls, not performing ACL manipulations` before and after displaying the protected document. This makes it sound like it is trying to hide the temp directory using ACLs, which obviously fails on filesystems that don't support them. In fact, the first message appears (suggesting that it has already extracted the protected files) *before* the license verification finishes. Oops.

When running the launcher again it doesn't prompt for the password (Looking in Wine's `regedit`, it stores the password in the registry key `HKEY_CURRENT_USER\Software\thelocker\6\41`. The `6` and `41` are the vendor and product IDs, which are displayed at the license prompt.). Interestingly, if the internet connection dies during the license verification, it will display an error message after a while, *but then show the protected document anyway even though it hasn't verified that the license is still active*! Oops.

------

The program does not seem to be distributed with a copy of the GPL or an offer of source code, and Googling `gnu site:bookguardpro.com` gives no results. They are clearly distributing a modified version of Sumatra PDF in binary form (there's no way vanilla Sumatra is going to support reading BGP-obfuscated PDFs, and I'm pretty sure it lets you select and open). Unless they have a special arrangement, this likely means they're in violation of Sumatra PDF's license (GPLv3). Oops.

# Verdict
Busted.

Oops.