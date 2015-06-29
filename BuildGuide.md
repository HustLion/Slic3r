
The user can't build slic3r with perl 5.16 or higher, recommend use Perl-5.14.4


Those people above used high version perl, such as the first one @megalomaniac with perl-5.16.0, which definitely can't work. And @bmcage used 5.18 I guess ( in this comment there is a line `cc -I/usr/lib/perl/5.18/CORE -DXS_VERSION="0.01" -DVERSION="0.01" -fPIC -xc++ `

Actually the newer version of Linux the new developers are using, the higher the possiblity they will encounter this problem and get upset. Because newer linux distributions ship perl 5.16 or even 5.18.

```` bash
curl -L http://xrl.us/perlbrewinstall | bash
source ~/perl5/perlbrew/etc/bashrc
perlbrew install 5.14.4
perlbrew switch perl-5.14.4
perlbrew install-cpanm
git clone https://github.com/alexrj/Slic3r.git
cd Slic3r
CPANM=/home/hustlion/perl5/perlbrew/bin/cpanm perl Build.PL
CPANM=/home/hustlion/perl5/perlbrew/bin/cpanm perl Build.PL --gui
./slic3r.pl
````

I have updated the wiki [page](https://github.com/alexrj/Slic3r/wiki/Running-Slic3r-from-git-on-GNU-Linux) accroding to this.

Hope this can be helpful :smile:.
