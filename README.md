# install-zabbix5-rocky8

#ORIGIN = brppzbm01 <br>
#DESTINY = brpldsinfzbs01

user postgres crontab
```bash
# Daily base backup of postgres databases
0 22 * * *  /var/lib/pgsql/pg_databasebackup.sh >> "/tmp/$(hostname).$(date +\%Y\%m\%d).pg_databasebackup.log" 2>&1
```
user root crontab
```bash
#Ansible: Rogier's DSG Report
5 * * * * /usr/local/report/dsg_linux_system_inventory.pl > /var/log/report/dsg_linux_system_inventory.log 2>&1
#
57 * * * * /data/dsg_linux_system_report/bin/refresh_cmdb.sh                                 > /data/dsg_linux_system_inventory/log/refresh_cmdb.log 2>&1
```


>> /var/lib/pgsql/pg_databasebackup.sh <<
```bash
#!/bin/bash

PORT_NUMBER=5432
BACKUP_DIR="/mnt/backup/OnDisk/brppzbm01/pgbackup"
EXITCODE=0

function pg_databasebackup ()
{

        #outpu start time
        echo "##############################################################################################################################"
        echo "START: $(date)"
        echo

        #get list of databases to backup
        DATABASES=$(psql -p $PORT_NUMBER -At -c "select datname from pg_database where not datistemplate and datallowconn order by datname;" postgres)

        #if no databases caputred then quit
        if [[ -z ${DATABASES} ]]
        then
                #error getting list of databses, set exit code and provide message
                EXITCODE=1
                echo "ERROR: No databases found, confirm port number: ${PORT_NUMBER} is correct and that cluster is online and accessible!!!"
        else
                #for each database
                for DATABASE in ${DATABASES}
                do
                        #if backup directory does not exist create it
                        if [ ! -d "${BACKUP_DIR}/${DATABASE}" ]; then mkdir "${BACKUP_DIR}/${DATABASE}"; fi

                        #set backup filename
                        BACKUPFILENAME="${BACKUP_DIR}/${DATABASE}/pg_dump.$(hostname).${PORT_NUMBER}.${DATABASE}.$(date +\%Y\%m\%d_\%H\%M\%S).custom"

                        #output backup command used
                        echo pg_dump -Fc -p ${PORT_NUMBER} "${DATABASE}" -f "${BACKUPFILENAME}"

                        #backup database
                        if ! pg_dump -Fc -p ${PORT_NUMBER} "${DATABASE}" -f "${BACKUPFILENAME}"
                        then
                                #error backing up, set exit code and provide message
                                EXITCODE=1
                                echo "ERROR: Backup operation for database ${DATABASE} unsuccessful!!!"
                        else
                                echo "INFO : Backup of database ${DATABASE} completed."

                                #delete old backup
                                #echo "INFO : Deleting following backups files older than ${DAYS_TO_RETAIN} days:"
                                #find "${BACKUP_DIR}/${DATABASE}" -maxdepth 1 -name "pg_dump.$(hostname).${PORT_NUMBER}.${DATABASE}.*.custom" -mtime +${DAYS_TO_RE$
                                #find "${BACKUP_DIR}/${DATABASE}" -maxdepth 1 -name "pg_dump.$(hostname).${PORT_NUMBER}.${DATABASE}.*.custom" -mtime +${DAYS_TO_RE$

                        fi
                        echo
                done
        fi


        #provide completion message
        if [ ${EXITCODE} -eq 0 ]; then
                echo "END  : $(date), SUCCESS"
        else
                echo "END  : $(date), ERRORS ENCOUNTERED!!! CHECK PREVIOUS MESSAGES!!!"
        fi

        exit ${EXITCODE}
}

pg_databasebackup;
```




>>/usr/local/report/dsg_linux_system_inventory.pl<<

```bash

#!/usr/bin/perl -w

use strict;

my      $version = '2018-05-29 RC1';
my      $indent  = "\t";

my      $dir    = $ARGV[0] || '/var/log/report';

my      $fn_tmp = sprintf "%s/%s", $dir, 'dsg_linux_system_inventory.tmp';
my      $fn_out = sprintf "%s/%s", $dir, 'dsg_linux_system_inventory.xml';

unless  (-d $dir)
{
        mkdir ($dir);
}

open    (OUTFILE, ">$fn_tmp");

printf  OUTFILE "<dsg_linux_system_inventory>\n";

system_inventory ($indent);
network_interfaces ($indent);
user_management ($indent);
sudo_config ($indent);
fs_config ($indent);
write_puppet ($indent);

printf  OUTFILE "</dsg_linux_system_inventory>\n";

close   (OUTFILE);

chmod   (0755, $dir);
chmod   (0644, $fn_tmp);

rename  ($fn_tmp, $fn_out);

$fn_tmp = sprintf "%s/%s", $dir, 'dsg_linux_system_inventory.patchlist.tmp';
$fn_out = sprintf "%s/%s", $dir, 'dsg_linux_system_inventory.patchlist.xml';

open    (OUTFILE, ">$fn_tmp");

printf  OUTFILE "<dsg_linux_system_inventory>\n";

system_inventory_patchlist ($indent);

printf  OUTFILE "</dsg_linux_system_inventory>\n";

close   (OUTFILE);

chmod   (0755, $dir);
chmod   (0644, $fn_tmp);

rename  ($fn_tmp, $fn_out);

$fn_tmp = sprintf "%s/%s", $dir, 'dsg_linux_system_inventory.packages.tmp';
$fn_out = sprintf "%s/%s", $dir, 'dsg_linux_system_inventory.packages.xml';

open    (OUTFILE, ">$fn_tmp");

printf  OUTFILE "<dsg_linux_system_inventory>\n";

system_inventory_packages ($indent);

printf  OUTFILE "</dsg_linux_system_inventory>\n";

close   (OUTFILE);

chmod   (0755, $dir);
chmod   (0644, $fn_tmp);

rename  ($fn_tmp, $fn_out);

sub     system_inventory
{
     my $prefix = shift;
     my $p2     = $prefix . $indent;

        printf  OUTFILE "%s<system>\n", $prefix;

        if      (-x '/bin/uname')
        {
             my $uname = `uname -a`; $uname =~ s/[\r\n]/ /g; $uname =~ s/ *$//;

                write_tag ($prefix, 'uname',    $uname);
        }
        else
        {
                write_tag ($prefix, 'error',    'Command /bin/uname not found');
        }

     my $hostname = `hostname`; $hostname =~ s/[\r\n]/ /g; $hostname =~ s/ *$//;
     my $time     = time;
     my $date     = `date -u '+\%_Y-\%m-\%d \%_H:\%_M'`; $date =~ s/[\r\n]/ /g; $date =~ s/ *$//;

        write_tag ($prefix, 'hostname',         $hostname);
        write_tag ($prefix, 'date',             $date);
        write_tag ($prefix, 'localtime',        format_time ($time));
        write_tag ($prefix, 'gmtime',           format_time ($time, 'gmtime'));
        write_tag ($prefix, 'report_version',   $version);

        show_hardware      ($p2, 'hardware');

        show_lsb_release   ($p2, 'redhat-release',      '/etc/redhat-release')  if      (-f '/etc/redhat-release');
        show_lsb_release   ($p2, 'os-release',          '/etc/os-release')      if      (-f '/etc/os-release');
        show_lsb_release   ($p2, 'lsb-release',         '/etc/lsb-release')     if      (-f '/etc/lsb-release');
        show_lsb_release_d ($p2)                                                if      (-d '/etc/lsb-release.d');

        printf  OUTFILE "%s</system>\n", $prefix;
}

sub     system_inventory_patchlist
{
     my $prefix = shift;
     my $p2     = $prefix . $indent;

        check_patchlist ($prefix);
}

sub     system_inventory_packages
{
     my $prefix = shift;
     my $p2     = $prefix . $indent;

        check_dpkg      ($prefix);
        check_yum       ($prefix);
}

sub     format_time
{
     my $time   = shift;
     my $zone   = shift;

     my ($sec,$min,$hour,$mday,$mon,$year,$wday,$yday,$isdst) = localtime($time);

        if      (defined ($zone) && ($zone eq 'gmtime'))
        {
                ($sec,$min,$hour,$mday,$mon,$year,$wday,$yday,$isdst) = gmtime($time);
        }

        return  sprintf "%04d-%02d-%02d %02d:%02d:%02d"
        ,       $year + 1900
        ,       $mon  + 1
        ,       $mday
        ,       $hour
        ,       $min
        ,       $sec;
}

sub     show_hardware
{
     my $prefix = shift;
     my $tag    = shift;

     my %data   = (); my $data  = \%data;

        $data->{mem_total}      = 0;
        $data->{mem_free}       = 0;
        $data->{swap_total}     = 0;
        $data->{swap_free}      = 0;
        $data->{buffers}        = 0;
        $data->{cache}          = 0;

     my $section = '?';

        open    (INFILE, "</proc/meminfo");

        while   (defined (my $line = <INFILE>))
        {
                chomp   ($line);

             my ($key, $value) = split ("[ \t]*:[ \t]*", $line);

             my $size   = $value;

                if      ($value =~ /[1-9][0-9]*\s{1,}kB$/)
                {
                        $size = $value; $size =~ s/ *kB$//; $size = sprintf "%.0f", $size * 1024;
                }

                if      (uc ($key) eq 'MEMTOTAL')       { $data->{mem_total}    += $size; }
                elsif   (uc ($key) eq 'MEMFREE')        { $data->{mem_free}     += $size; }
                elsif   (uc ($key) eq 'SWAPTOTAL')      { $data->{swap_total}   += $size; }
                elsif   (uc ($key) eq 'SWAPFREE')       { $data->{swap_free}    += $size; }
                elsif   (uc ($key) eq 'BUFFERS')        { $data->{buffers}      += $size; }
                elsif   (uc ($key) eq 'CACHED')         { $data->{cache}        += $size; }
        }

        close   (INFILE);

     my %cpu    = (); my $cpu = \%cpu;
     my $proc;

        open    (INFILE, "</proc/cpuinfo");

        while   (defined (my $line = <INFILE>))
        {
                chomp   ($line);

             my ($key, $value) = split ("[ \t]*:[ \t]*", $line);

                $key    = $key   || 'X';
             my $val    = $value || '';

                if      ($val =~ /[1-9][0-9]*\s{1,}kB$/)
                {
                        $val = $value; $val =~ s/ *kB$//; $val = sprintf "%.f0f", $val * 1024;
                }
                elsif   (uc ($key) eq 'CPU MHZ')
                {
                        $val = $value; $val =~ s/ *kB$//; $val = sprintf "%.f0f", $val * 1024 * 1024;
                }

                if      (uc ($key) eq 'PROCESSOR')
                {
                     my $id     = $value; if ($id =~ /^[0-9][0-9]*$/) { $id = sprintf "%010d", $id; }
                     my %proc   = (); $proc     = $cpu{$id} = \%proc;
                }
                elsif   (uc ($key) eq 'MODEL NAME')     { $proc->{model_name}   = $val; }
                elsif   (uc ($key) eq 'CPU MHZ')        { $proc->{speed}        = $val; }
                elsif   (uc ($key) eq 'CACHE SIZE')     { $proc->{cache}        = $val; }
        }

        close   (INFILE);

     my $p      = $prefix . $indent;
     my $p1     = $p      . $indent;

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<%s>\n",     $prefix, $tag;
        printf  OUTFILE "%s<%s>\n",     $p,      'memory';

        write_tag ($p, 'physical',      $data->{mem_total});
        write_tag ($p, 'swap',          $data->{swap_total});

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<%s>\n",     $p1, 'free';

        write_tag ($p1, 'physical',     $data->{mem_free});
        write_tag ($p1, 'swap',         $data->{swap_free});
        write_tag ($p1, 'buffers',      $data->{buffers});
        write_tag ($p1, 'cache',        $data->{cache});

        printf  OUTFILE "%s</%s>\n",    $p1, 'free';
        printf  OUTFILE "%s</%s>\n",    $p,  'memory';

        foreach my $id (sort keys %{$cpu})
        {
             my $proc   = $cpu{$id};
                printf  OUTFILE "\n";
                printf  OUTFILE "%s<%s>\n",     $p, 'processor';

                write_tag ($p, 'model_name',    $proc->{model_name});
                write_tag ($p, 'speed', $proc->{speed});
                write_tag ($p, 'cache', $proc->{cache});

                printf  OUTFILE "%s</%s>\n",    $p, 'processor';
        }

        printf  OUTFILE "%s</%s>\n",    $prefix, $tag;
}

sub     show_lsb_release
{
     my $prefix = shift;
     my $tag    = shift;
     my $fn     = shift;

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<%s>\n",  $prefix, $tag;

        open    (INFILE, "<$fn");

        while   (defined (my $line = <INFILE>))
        {
                $line   =~ s/[\r\n]//g;

                if      ($line =~ /^[^#]*=/)
                {
                     my ($key, $value) = split (' *= *', $line);

                        $value =~ s/ *$//;

                        if      ($value =~ /^".*"$/)
                        {
                                $value = substr ($value, 1, length ($value) - 2);
                        }

                        write_tag ($prefix, lc ($key),  $value);
                }
                elsif   ($line !~ /^ *$/)
                {
                        write_tag ($prefix, 'free_format',      $line);
                }
        }

        close   (INFILE);

        printf  OUTFILE "%s</%s>\n", $prefix, $tag;
}

sub     show_lsb_release_d
{
     my $prefix = shift;

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<lsb-release.d>\n",  $prefix;
        printf  OUTFILE "%s</lsb-release.d>\n", $prefix;
}

sub     check_patchlist
{
     my $prefix = shift;
     my $p2     = $prefix . $indent;

     my $list   = '/var/lib/dsg/patchlist.csv';
     my $idx    = 0;

        return  unless   (-f $list);

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<patchlist>\n", $prefix;

        open    (INFILE, "<$list");

        while   (defined (my $line = <INFILE>))
        {
                $line =~ s/[\r\n]//g;

             my ($hostname, $method, $pkg, $version, $available, $date) = split (';', $line);

                if      (++$idx == 1)
                {
                        printf  OUTFILE "%s<header>\n", $p2;

                        write_tag ($p2, 'hostname',     $hostname);
                        write_tag ($p2, 'method',       $method);
                        write_tag ($p2, 'datetime',     $date);

                        printf  OUTFILE "%s</header>\n", $p2;
                }

                printf  OUTFILE "\n";
                printf  OUTFILE "%s<package>\n", $p2;

                write_tag ($p2, 'name',         $pkg);
                write_tag ($p2, 'version',      $version);
                write_tag ($p2, 'available',    $available);

                printf  OUTFILE "%s</package>\n", $p2;
        }

        close   (INFILE);

        printf  OUTFILE "%s</patchlist>\n", $prefix;
}

sub     check_dpkg
{
     my $prefix = shift;
     my $p2     = $prefix . $indent;

     my $dpkg   = '/usr/bin/dpkg';

        return  unless   (-x $dpkg);

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<dpkg>\n", $prefix;

        open    (INFILE, "$dpkg -l |");

     my $hdr1   = <INFILE>; $hdr1 =~ s/[\r\n]//g;
     my $hdr2   = <INFILE>; $hdr2 =~ s/[\r\n]//g;
     my $hdr3   = <INFILE>; $hdr3 =~ s/[\r\n]//g;
     my $hdr4   = <INFILE>; $hdr4 =~ s/[\r\n]//g;
     my $hdr5   = <INFILE>; $hdr5 =~ s/[\r\n]//g;
     my $cnt    = 0;

        unless  ($hdr5 =~ /^\+\+\+-/)
        {
                write_tag ($prefix, 'error',    'Unexpected output format');

                printf  OUTFILE "\n";

                write_tag ($prefix, 'line',     $hdr1);
                write_tag ($prefix, 'line',     $hdr2);
                write_tag ($prefix, 'line',     $hdr3);
                write_tag ($prefix, 'line',     $hdr4);
                write_tag ($prefix, 'line',     $hdr5);

                printf  OUTFILE "%s</dpkg>\n",  $prefix;

                return;
        }

        while   (defined (my $line = <INFILE>))
        {
                if      (++$cnt > 1)
                {
                        printf  OUTFILE "\n";
                }

             my $pkg    = split_line_with_header ($hdr5, '-', $hdr4, $line);

                printf  OUTFILE "%s<package>\n",        $p2;

#               write_tag ($p2, 'fields',       join (', ', sort (keys (%{$pkg}))));
                write_tag ($p2, 'name',         $pkg->{name});
                write_tag ($p2, 'description',  $pkg->{description});
                write_tag ($p2, 'version',      $pkg->{version});
                write_tag ($p2, 'architecture', $pkg->{architecture});
                write_tag ($p2, 'status',       $pkg->{status});
                write_tag ($p2, 'desired',      $pkg->{desired});
                write_tag ($p2, 'error',        $pkg->{error});

                printf  OUTFILE "%s</package>\n",       $p2;
        }

        close   (INFILE);

        printf  OUTFILE "%s</dpkg>\n", $prefix;
}

sub     check_yum
{
     my $prefix = shift;
     my $p2     = $prefix . $indent;

     my $yum    = '/usr/bin/yum';

        if      (-f  '/usr/bin/yum')    { $yum = '/usr/bin/yum'; }
        elsif   (-f  '/bin/yum')        { $yum = '/bin/yum'; }

        return  unless   (-x $yum);

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<yum>\n", $prefix;

        open    (INFILE, "$yum list installed |");

     my $cnt    = -1;

        while   (defined (my $line = <INFILE>))
        {
                $line   =~ s/[\r\n]//g;

                if      ($cnt < 0)
                {
                        $cnt    = 0     if ($line =~ /^Installed Packages/);
                }
                else
                {
                        printf  OUTFILE "\n" if (++$cnt > 1);
                        printf  OUTFILE "%s<package>\n", $p2;

                     my ($name, $version, $status) = split ("  *", $line);

                        unless  (defined ($status))
                        {
                             my $l2     = <INFILE>; $l2 =~ s/[\r\n]//g;

                                ($name, $version, $status) = split ("  *", "$line $l2");;
                        }

                        write_tag ($p2, 'name',         $name);
                        write_tag ($p2, 'version',      $version);
                        write_tag ($p2, 'status',       $status);

                        printf  OUTFILE "%s</package>\n", $p2;
                }
        }

        close   (INFILE);

        printf  OUTFILE "%s</yum>\n", $prefix;
}

sub     split_line_with_header
{
     my $marker         = shift; $marker =~ s/[\r\n]//;
     my $separator      = shift;
     my $labels         = shift; $labels =~ s/[\r\n]//;
     my $line           = shift; $line   =~ s/[\r\n]//;

     my %result         = ();

        foreach my $field (split ($separator, $marker))
        {
             my $label  = substr ($labels, 0, length ($field)); $label =~ s/ *$//;
             my $value  = substr ($line,   0, length ($field)); $value =~ s/ *$//;

#               printf  "Field : %s (%s)\n",     $field, length ($field);
#               printf  "Label : %s (%s, %s)\n", $label, length ($labels), $labels;
#               printf  "Value : %s (%s, %s)\n", $value, length ($line),   $line;

                if      (length ($labels) > length ($field) + length ($separator))
                {
                        $labels = substr ($labels, length ($field) + length ($separator));
                        $line   = substr ($line,   length ($field) + length ($separator));
                }

                if      ($label eq '||/')
                {
                        $result{desired} = substr ($value, 0, 1);
                        $result{status}  = substr ($value, 1, 1);
                        $result{error}   = substr ($value, 2, 1);
                }
                else
                {
                        $result{lc($label)} = $value;
                }
        }

        return  \%result
}

sub     network_interfaces
{
     my $prefix = shift;

     my $p      = $prefix . $indent;
     my $p1     = $p      . $indent;
     my $p2     = $p1     . $indent;
     my $p3     = $p2     . $indent;

     my $dev;

     my %dev_by_nr      = ();
     my %dev_by_name    = ();

        foreach my $line (split ("\n", `/sbin/ip addr`))
        {

                if      ($line =~ /^[0-9][0-9]*:/)
                {
                     my %dev    = (); $dev = \%dev;

                        ($dev->{nr}, $dev->{name}, $dev->{flags}) = split (': ', $line);

                     my $key = sprintf "%04d", $dev->{nr};

                        $dev_by_nr{$key} = $dev_by_name{$dev->{name}} = $dev;

                     my @flags = split ('  *', $dev->{flags});

                        while   (defined (my $flag = shift (@flags)))
                        {
                                if      ($flag eq 'mtu')        { $dev->{mtu}    = shift (@flags); }
                                elsif   ($flag eq 'master')     { $dev->{master} = shift (@flags); }
                                elsif   ($flag eq 'state')      { $dev->{state}  = shift (@flags); }
                        }
                }
                elsif   (defined ($dev))
                {
                        if      ($line =~ /^ *inet /)
                        {
                             my %ip = (); my $ip = \%ip;

                             my $ip_string = $line; $ip_string =~ s/^ *inet *//; $ip_string =~ s/ .*//;

                                ($ip->{address}, $ip->{subnet_class}) = split ('/', $ip_string);

                                if      ($line =~ / secondary /)
                                {
                                        $ip->{secondary} = 1;
                                }

                                unless  (defined ($dev->{ip}))
                                {
                                     my %i = (); $dev->{ip} = \%i;
                                }

                                $dev->{ip}->{$ip->{address}} = $ip;
                        }
                }
        }

     my $nr     = 0;

        foreach my $line (split ("\n", `/sbin/ip route list`))
        {
             my $dest   = lc $line; $dest =~ s/ .*//;
             my $dev    = lc $line; $dev  =~ s/.* dev *//; $dev =~ s/ .*//;

             my $device = $dev_by_name{$dev};

                unless  (defined ($device))
                {
                     my $key    = sprintf "9%03d", $nr;
                     my %d      = (); $device = $dev_by_nr{$key} = $dev_by_name{$dev} = \%d;
                }

                unless  (defined ($device->{route}))
                {
                     my %r = (); $device->{route} = \%r;
                }

             my %route  = (); my $route = $device->{route}->{$dest} = \%route;

                $route->{destination}   = $dest;
                $route->{device}        = $dev;

                if      ($line =~ / via /)
                {
                     my $gateway = $line; $gateway =~ s/.* via *//; $gateway =~ s/ .*//;

                        $route->{gateway} = $gateway;
                }

                if      ($line =~ / src /)
                {
                     my $src = $line; $src =~ s/.* src *//; $src =~ s/ .*//;

                        $route->{src} = $src;
                }
        }

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<network>\n", $prefix;

     my $dev_idx = 0;

        foreach my $key (sort keys %dev_by_nr)
        {
                if      (++$dev_idx > 1)
                {
                        printf  OUTFILE "\n";
                }

                printf  OUTFILE "%s<interface>\n", $p;

             my $device = $dev_by_nr{$key};

                write_tag ($p, 'name',          $device->{name});
                write_tag ($p, 'number',        $device->{nr});
                write_tag ($p, 'master',        $device->{master});
                write_tag ($p, 'state',         $device->{state});
                write_tag ($p, 'mtu',           $device->{mtu});

                if      (defined ($device->{ip}))
                {
                        foreach my $ip_address (sort keys %{$device->{ip}})
                        {
                             my $ip     = $device->{ip}->{$ip_address};

                                printf  OUTFILE "\n";
                                printf  OUTFILE "%s<ip>\n", $p1;

                                write_tag ($p1, 'address',      $ip_address);
                                write_tag ($p1, 'subnet_class', $ip->{subnet_class});
                                write_tag ($p1, 'secondary',    $ip->{secondary});

                                printf  OUTFILE "%s</ip>\n", $p1;
                        }
                }

                if      (defined ($device->{route}))
                {
                        foreach my $destination (sort keys %{$device->{route}})
                        {
                             my $route  = $device->{route}->{$destination};

                                printf  OUTFILE "\n";
                                printf  OUTFILE "%s<route>\n", $p1;

                                write_tag ($p1, 'destination',  $destination);
                                write_tag ($p1, 'gateway',      $route->{gateway});
                                write_tag ($p1, 'src',          $route->{src});

                                printf  OUTFILE "%s</route>\n", $p1;
                        }
                }

                printf  OUTFILE "%s</interface>\n", $p;
        }

        printf  OUTFILE "%s</network>\n", $prefix;
}

sub     user_management
{
     my $prefix = shift;

     my $p      = $prefix . $indent;
     my $p1     = $p      . $indent;
     my $p2     = $p1     . $indent;
     my $p3     = $p2     . $indent;

     my %users  = (); my $users = \%users;

        open    (INFILE, "</etc/passwd");

        while   (defined (my $line = <INFILE>))
        {
                $line   =~ s/[\r\n]//g;

             my ($uname, $pwd, $uid, $gid, $name, $homedir, $shell) = split (':', $line);

             my %user = (); my $user = $users->{$uname} = \%user;

                $user->{uname}  = $uname;
                $user->{name}   = $name;
                $user->{pwd}    = $pwd;
                $user->{uid}    = $uid;
                $user->{gid}    = $gid;
                $user->{home}   = $homedir;
                $user->{shell}  = $shell;

#               printf  OUTFILE "%s<user>%s</user>\n", $prefix, xml_escape ($uname);
        }

        close   (INFILE);

#       printf  OUTFILE "\n";

        open    (INFILE, "</etc/shadow");

        while   (defined (my $line = <INFILE>))
        {
                $line   =~ s/[\r\n]//g;

             my ($uname, $pwd, $pw_change_date, $pw_changable_date, $pw_expires_date,
                 $pw_warn_days, $pw_grace_days, $disabled_date, $reserved) = split (':', $line);

             my $user = $users->{$uname};

                if      (defined ($user))
                {
                        $user->{has_shadow_entry}       = 1;
                        $user->{password}               = $pwd; # This should contain the real password
                }

#               printf  OUTFILE "%s\n", $line;
        }

        close   (INFILE);

     my $cnt    = 0;

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<user_management>\n", $prefix;

        open    (INFILE, "</etc/group");

        while   (defined (my $line = <INFILE>))
        {
                $line   =~ s/[\r\n]//g; $line =~ s/ *$//;

                if      ($line eq '')
                {
                        # A warning might be appropriate here, probably manually edited.
                }
                else
                {
                     my ($group, $pwd, $gid, $members) = split (':', $line);

                        printf  OUTFILE "\n" if (++$cnt > 1);
                        printf  OUTFILE "%s<group>\n", $p;

                        write_tag ($p, 'name',  $group);
                        write_tag ($p, 'gid',   $gid);

                        unless  ($members eq '')
                        {
                                printf  OUTFILE "\n";

                                foreach my $member (split (' *, *', $members))
                                {
                                        write_tag ($p, 'member',        $member);
                                }
                        }

                        printf  OUTFILE "%s</group>\n", $p;
                }
        }

        close   (INFILE);

        foreach my $uname (sort keys %users)
        {
             my $user = $users->{$uname};

                printf  OUTFILE "\n";
                printf  OUTFILE "%s<user>\n", $p;

             my $pwd    = $user->{pwd};

                if      ($pwd eq 'x')
                {
                        if      ($user->{has_shadow_entry})
                        {
                                $pwd    = $user->{password};
                        }
                }

             my $status = 'enabled';

                if      ($pwd eq '')    { $status       = 'no password required'; }
                elsif   ($pwd eq '!')   { $status       = 'password disabled'; }
                elsif   ($pwd eq '!!')  { $status       = 'password disabled'; }
                elsif   ($pwd eq '*')   { $status       = 'account disabled'; }

                write_tag ($p, 'username',      $uname);
                write_tag ($p, 'name',          $user->{name});
                write_tag ($p, 'uid',           $user->{uid});
                write_tag ($p, 'gid',           $user->{gid});
                write_tag ($p, 'pwd',           $status);

             my $home = $user->{home};

                unless  ($home eq '')
                {
                        printf  OUTFILE "\n";
                        printf  OUTFILE "%s<home>\n",  $p1;

                        write_tag ($p1, 'directory',    $user->{home});

                        if      (defined (my $hstat = fstat ($home)))
                        {
                                write_tag ($p1, 'owner',                $hstat->{uid});
                                write_tag ($p1, 'group',                $hstat->{gid});
                                write_tag ($p1, 'permissions',  $hstat->{permissions});
                        }
                        else
                        {
                                write_tag ($p1, 'permissions', 'directory does not exist');
                        }

                     my $ssh    = "$home/.ssh";

                        if      (defined (my $sstat = fstat ($ssh)))
                        {
                                printf  OUTFILE "\n";
                                printf  OUTFILE "%s<sshdir>\n",  $p2;

                                write_tag ($p2, 'owner',        $sstat->{uid});
                                write_tag ($p2, 'group',        $sstat->{gid});
#                               write_tag ($p2, 'type',         $sstat->{type});                # '040000' is a directory'
                                write_tag ($p2, 'permissions',  $sstat->{permissions});

                                foreach my $filename ('authorized_keys', 'authorized_keys2')
                                {
                                        show_keyfile ($p3, $ssh, $filename);
                                }

                                printf  OUTFILE "%s</sshdir>\n", $p2;
                        }

                        printf  OUTFILE "%s</home>\n", $p1;
                }

                printf  OUTFILE "%s</user>\n", $p;
        }

        printf  OUTFILE "%s</user_management>\n", $prefix;
}

sub     show_keyfile
{
     my $prefix         = shift;
     my $dir            = shift;
     my $filename       = shift;

     my $p2             = $prefix . $indent;
     my $fn             = "$dir/$filename";

        if      (defined (my $s = fstat ($fn)))
        {
                printf  OUTFILE "\n";
                printf  OUTFILE "%s<file>\n",   $prefix;

                write_tag ($prefix, 'name',             $filename);
                write_tag ($prefix, 'owner',            $s->{uid});
                write_tag ($prefix, 'group',            $s->{gid});
#               write_tag ($prefix, 'type',             $s->{type});            # '100000' is a file
                write_tag ($prefix, 'permissions',      $s->{permissions});

                if      ($s->{type} eq '100000')
                {
                        open    (INFILE, "<$fn");
                        my $key_found = 0;

                        while   (defined (my $line = <INFILE>))
                        {
                                $line   =~ s/[\r\n]//g; $line =~ s/#.*//; $line =~ s/^ *//; $line =~ s/ *$//;

                                unless  (($line eq '') || ($key_found == 1))
                                {
                                     my ($type, $key, $comment) = split ('  *', $line);

                                        printf  OUTFILE "\n";
                                        printf  OUTFILE "%s<public_key>\n",     $p2;

                                        write_tag ($p2, 'type',         $type);
                                        write_tag ($p2, 'key',          $key);
                                        write_tag ($p2, 'comment',      $comment);

                                        printf  OUTFILE "%s</public_key>\n",    $p2;

                                        $key_found = 1;
                                }
                        }

                        close   (INFILE);
                }
                else
                {
                     my $error  = sprintf "Not a regular file: Type is '%s'", $s->{type};

                        write_tag ($prefix, 'error', $error);
                }

                printf  OUTFILE "%s</file>\n",  $prefix;
        }
}

sub     sudo_config
{
     my $prefix = shift;

     my $p      = $prefix . $indent;
     my $idx    = 0;

        printf  OUTFILE "\n%s<sudo_config>\n", $prefix;

        open    (INFILE, "</etc/group");

        $idx    = show_file ('/etc/sudo.conf',  $p, $idx);
        $idx    = show_file ('/etc/sudoers',    $p, $idx);

        if      (-d '/etc/sudoers.d')
        {
             my $dh; opendir ($dh, '/etc/sudoers.d');

                while   (defined (my $filename = readdir ($dh)))
                {
                     my $fn     = sprintf "/etc/sudoers.d/%s", $filename;

                        unless  (-d $fn)
                        {
                                $idx = show_file ($fn, $p, $idx);
                        }
                }

                closedir ($dh);
        }

        printf  OUTFILE "%s</sudo_config>\n", $prefix;
}

sub     fs_config
{
     my $prefix = shift;

     my %fsdata = (); my $fsdata = \%fsdata;
     my %fstab  = (); my $fstab  = \%fstab;

        $fsdata->{device}        = $fstab;
        $fsdata->{uuid}          = get_links ($fsdata, '/dev/disk/by-uuid');
        $fsdata->{label}         = get_links ($fsdata, '/dev/disk/by-label');

        open    (INFILE, "</etc/mtab");

        while   (defined (my $line = <INFILE>))
        {
                if      (defined (my $d = parse_line ($fsdata, $line)))
                {
                        $d->{device}->{mtab} = $d->{config};
                }
        }

        close   (INFILE);

        open    (INFILE, "</etc/fstab");

        while   (defined (my $line = <INFILE>))
        {
                if      (defined (my $d = parse_line ($fsdata, $line)))
                {
                        $d->{device}->{fstab} = $d->{config};
                }
        }

        close   (INFILE);

#       printf  "\n%s\n", Dumper ($fstab);

        compare_fstab ($prefix, $fsdata);
}

sub     get_links
{
     my $fsdata = shift;
     my $dir    = shift;
     my $dh;

     my %data   = (); my $data = \%data;

        if      (-d $dir)
        {
                opendir  ($dh, $dir);

                while   (defined (my $filename = readdir ($dh)))
                {
                        if      (-l "$dir/$filename")
                        {
                             my $dev    = resolve_symlinks ("$dir/$filename");

#                               printf  "%-12s %s\n", $dev, $filename;

                             my ($d, $method) = get_device ($fsdata, $dev);

                                $data->{"$dir/$filename"} = $d;

                                if      ($dir =~ /by-uuid/)
                                {
                                        $d->{uuid} = $filename;

                                        $data->{$filename} = $d;
                                }
                                elsif   ($dir =~ /by-label/)
                                {
                                        $d->{label} = $filename;

                                        $data->{$filename} = $d;
                                }
                        }
                }

                closedir ($dh);
        }

        return  $data;
}

sub     get_device
{
     my $fsdata = shift;
     my $name   = shift;
     my $d;

        if      ($name =~ /^UUID=/)
        {
             my $id     = $name; $id =~ s/^UUID=//;

                if      (defined ($d = $fsdata->{uuid}->{$id}))
                {
                        return  ($d, 'uuid');
                }
        }
        elsif   ($name =~ /^LABEL=/)
        {
             my $label  = $name; $label =~ s/^LABEL=\/*//;

                if      (defined ($d = $fsdata->{label}->{$label}))
                {
                        return  ($d, 'label');
                }
        }
        else
        {
                $name   = resolve_device ($name);

                if      (defined ($d = $fsdata->{device}->{$name}))
                {
                        return  ($d, 'direct');
                }
        }

     my %d      = (); $d = \%$d;

        $fsdata->{device}->{$name}      = $d;
        $d->{name}                      = $name;

        return  ($d, 'direct');
}

sub     parse_line
{
     my $fsdata = shift;
     my $line   = shift;
        $line   =~ s/[\r\n]//g;

        if      (($line =~ /^#/) || ($line =~ /^\s*$/))
        {
                return  undef; # Ignore empty lines and comments.
        }

     my @ignore_devices = ('none', 'tmpfs', 'udev', 'proc', 'rootfs'
        ,       'systemd', 'systemd-1'
        ,       'devtmpfs', 'configfs', 'sunrpc', 'rpc_pipefs'
        ,       'sysfs', 'systemd-1', 'sysfs', 'autofs', 'securityfs', 'debugfs', 'selinuxfs', 'hugetlbfs'
        ,       'devpts', 'pstore', 'binfmt_misc', 'fusectl', 'cgroup'
        ,       'mqueue'
        );

        foreach my $d (@ignore_devices)
        {
                if      ($line =~ /^${d}\s/)
                {
                        return  undef; # Ignore pseudo-filesystems
                }
        }

        $line   =~ s/^\s*//; $line =~ s/\s*$//; $line =~ s/\s\s*/ /g;

     my ($name, $mount_point, $type, $options, $dump, $pass) = split (' ', $line);

#       printf  "%s : %s %s %s %s\n", $line, $name, $mount_point, $type, $options;

     my %d = (); my $d = \%d;
     my %c = (); my $c = \%c;

        ($d->{device}, $c->{method}) = get_device ($fsdata, $name);

        $d->{config}            = $c;
        $c->{mount_point}       = $mount_point;
        $c->{fs_type}           = $type;
        $c->{options}           = $options;
        $c->{path}              = $name;

#       printf  "\n%s\n\n", Dumper ($d);

        return  $d;
}

sub     resolve_symlinks
{
     my $fn  = shift;
     my $idx = 0;

        while   (-l $fn)
        {
             my $new    = readlink ($fn);
             my $dev    = $new;

             my $dir    = $fn; $dir =~ s/[^\/]*$//;

                unless  ($dev =~ /^\//)
                {
                        $dev    = sprintf "%s%s", $dir, $dev;

                        while   ($dev =~ /\/[a-z][a-z\-\_]*\/\.\.\//)
                        {
                                $dev =~ s/\/[a-z][a-z\-\_]*\/\.\.\//\//;
                        }
                }

#               printf  "%s => %s => %s\n", $fn, $new, $dev;

                $fn     = $dev;

                if      (++$idx > 10)
                {
#                       die "Recursion error";

                        return  $fn;
                }
        }

        return  $fn;
}

sub     resolve_device
{
     my $fn     = shift;

        $fn     = resolve_symlinks ($fn);

        if      (-b $fn)
        {
             my $s = fstat ($fn);

                return  $s->{block_device};
        }
        else
        {
#               printf  "%s is not a valid block device.\n", $fn;

                return  $fn;
        }
}

sub     compare_fstab
{
     my $prefix         = shift;
     my $fsdata         = shift;

     my $p              = $prefix . $indent;
     my $p2             = $p      . $indent;
     my $p3             = $p2     . $indent;

     my $cnt            = 0;
     my $errors         = 0;
     my $warnings       = 0;

        printf  OUTFILE "\n";
        printf  OUTFILE "%s<fs_config>\n", $prefix;

        foreach my $dname (sort keys %{$fsdata->{device}})
        {
             my $d      = $fsdata->{device}->{$dname};

                if      (++$cnt > 1)
                {
                        printf  OUTFILE "\n";
                }

                printf  OUTFILE "%s<device>\n", $p;

                write_tag ($p, 'id', $dname);

                if      (defined (my $u = $d->{label}))         { write_tag ($p, 'label',  $u); }
                if      (defined (my $u = $d->{uuid}))          { write_tag ($p, 'uuid',   $u); }
                if      (defined (my $u = $d->{mapper}))        { write_tag ($p, 'mapper', $u); }

                if      (defined (my $t = $d->{fstab}))
                {
                        printf  OUTFILE "\n";
                        printf  OUTFILE "%s<fstab>\n", $p2;

                        write_tag ($p2, 'mount_point',  $t->{mount_point});
                        write_tag ($p2, 'path',         $t->{path});
                        write_tag ($p2, 'fs_type',      $t->{fs_type});
                        write_tag ($p2, 'options',      $t->{options});
                        write_tag ($p2, 'method',       $t->{method});

                        printf  OUTFILE "%s</fstab>\n", $p2;
                }

                if      (defined (my $t = $d->{mtab}))
                {
                        printf  OUTFILE "\n";
                        printf  OUTFILE "%s<mtab>\n", $p2;

                        write_tag ($p2, 'mount_point',  $t->{mount_point});
                        write_tag ($p2, 'path',         $t->{path});
                        write_tag ($p2, 'fs_type',      $t->{fs_type});
                        write_tag ($p2, 'options',      $t->{options});
                        write_tag ($p2, 'method',       $t->{method});

                        printf  OUTFILE "%s</mtab>\n",  $p2;
                }

             my $error;
             my $warning;

                if      (defined (my $f = $d->{fstab}) && defined (my $m = $d->{mtab}))
                {
                        if      ($f->{mount_point} ne $m->{mount_point})
                        {
                                $error  = 'Different mountpoint in fstab than in mtab';
                        }

                        if      ($f->{fs_type} ne $m->{fs_type})
                        {
                                $error  = 'Different filesystem type in fstab than in mtab';
                        }
                }
                elsif   (defined ($d->{fstab}))
                {
                        if      ($d->{fstab}->{mount_point} eq 'none')
                        {
                        }
                        elsif   ($d->{fstab}->{mount_point} eq 'swap')
                        {
                        }
                        else
                        {
                                $error  = 'Entry is in fstab, but not in mtab';
                        }
                }
                elsif   (defined ($d->{mtab}))
                {
                        $warning        = 'Entry is in mtab, but not in fstab';
                }

                if      (defined ($error))      { printf OUTFILE "\n"; write_tag ($p, 'error',   $error);   $errors++;   }
                elsif   (defined ($warning))    { printf OUTFILE "\n"; write_tag ($p, 'warning', $warning); $warnings++; }

                printf  OUTFILE "%s</device>\n", $p;
        }

        printf  OUTFILE "\n";

        if      ($errors > 0)
        {
                write_tag ($prefix, 'fs_status',  'ERROR');
                write_tag ($prefix, 'fs_summary', "$errors error(s), $warnings warning(s) found");
        }
        elsif   ($warnings > 0)
        {
                write_tag ($prefix, 'fs_status',  'WARNING');
                write_tag ($prefix, 'fs_summary', "$warnings warning(s) found");
        }
        else
        {
                write_tag ($prefix, 'fs_status',  'OK');
        }

        printf  OUTFILE "%s</fs_config>\n", $prefix;
}

sub     show_file
{
     my $fn     = shift;
     my $prefix = shift;
     my $idx    = shift;

     my $p      = $prefix . $indent;
     my $lines  = 0;

        unless  (-r $fn)
        {
                return  $idx;
        }

        unless  ($idx == 0)
        {
                printf OUTFILE "\n";
        }

        printf  OUTFILE "%s<file>\n", $prefix;

        write_tag ($p, 'filename',      $fn);

        if      (open (CONFIG_FILE, "<$fn"))
        {
                while   (defined (my $line = <CONFIG_FILE>))
                {
                        $line   =~ s/[ \r\n]*$//;

                        if      (++$lines == 1)
                        {
                                printf  OUTFILE "\n";
                        }

                        write_tag ($p, 'line',  $line);
                }

                close   (CONFIG_FILE);
        }
        else
        {
                write_tag ($p, 'error', 'Unable to read file');
        }

        printf  OUTFILE "%s</file>\n", $prefix;

        return  $idx + 1;
}

sub     fstat
{
    my  $fn     = shift;
    my  %s      = ();

        (       $s{dev},        $s{inode},      $s{mode},       $s{nlink}
        ,       $s{uid},        $s{gid},        $s{rdev},       $s{size}
        ,       $s{atime},      $s{mtime},      $s{ctime},
        ,       $s{blksize},    $s{blocks})                     = stat ($fn);

        return  undef unless (defined ($s{dev}));

        if      (-b $fn)
        {
             my $major  = int ($s{rdev}/256);
             my $minor  = $s{rdev} - (256 * $major);

                $s{block_device} = sprintf "%03d:%03d", $major, $minor;
        }

     my $perm   = $s{mode} & 07777;
     my $type   = $s{mode} - $perm;

        $s{permissions} = sprintf "%04o", $perm;
        $s{type}        = sprintf "%06o", $type;

        return  \%s;
}

sub     write_tag
{
     my $prefix = shift;
     my $tag    = shift;
     my $value  = shift;
     my $force  = shift;

     my $v      = xml_escape ($value);

        if      ($v eq '')
        {
                if      (defined ($force))
                {
                        printf  OUTFILE "%s%s<%s/>\n", $prefix, $indent, $tag;
                }
        }
        else
        {
                printf  OUTFILE "%s%s<%s>%s</%s>\n", $prefix, $indent, $tag, xml_escape ($value), $tag;
        }
}

sub     xml_escape
{
     my $text   = shift;

        return  ''      unless  (defined ($text));
        return  ''      if      ($text =~ /^ *$/);

        $text   =~ s/&/&amp;/g;
        $text   =~ s/</&lt;/g;
        $text   =~ s/>/&gt;/g;

        return  $text;
}

sub     write_puppet
{
     my $prefix = shift;

     my $p      = $prefix . $indent;

        printf  OUTFILE "\n%s<puppet>\n", $prefix;

     my $config3        = '/etc/puppet/puppet.conf';
     my $config4        = '/etc/puppetlabs/puppet/puppet.conf';

     my $fn_real        = '/opt/puppetlabs/puppet/cache/state/last_run_summary.yaml';
     my $fn_noop        = '/opt/puppetlabs/puppet/cache/state.noop/last_run_summary.yaml';

     my $version        = 'None';
     my $server;
     my $environment;

        if      (-f $config4)
        {
                if      (-f $config3)   { $version = 'Frankenstein'; }
                else                    { $version = 'Puppet4'; }

                ($server, $environment) = read_puppet_conf ($config4);
        }
        elsif   (-f $config3)
        {
                $version                = 'Puppet3';
                ($server, $environment) = read_puppet_conf ($config3);
        }

        # Note: All tags are unique (and by consequence, longer than necessary.
        # This is to enable easy value extraction using oneliners (Zabbix), without an XML parser.

        write_tag ($p, 'puppet_version',        $version);
        write_tag ($p, 'puppet_server',         $server);
        write_tag ($p, 'puppet_environment',    $environment);

        if      ((-r $fn_real) && defined (my $yaml = read_yaml ($fn_real)))
        {
             my $d      = flatten_node ($yaml);
             my $age    = time() - $d->{time}->{last_run};

                printf  OUTFILE "\n";

                write_tag ($p, 'puppet_last_run',       $d->{time}->{last_run});
                write_tag ($p, 'puppet_age',            $age);
                write_tag ($p, 'puppet_runtime',        $d->{time}->{total});
                write_tag ($p, 'puppet_changes',        $d->{changes}->{total});
                write_tag ($p, 'puppet_success',        $d->{events}->{success});
                write_tag ($p, 'puppet_failures',       $d->{events}->{failure});
        }
        else
        {
                printf  OUTFILE "\n";

                write_tag ($p, 'warning', "File '$fn_real' not found for main puppet run.");
        }

        if      ((-r $fn_noop) && defined (my $yaml = read_yaml ($fn_noop)))
        {
             my $d      = flatten_node ($yaml);
             my $age    = time() - $d->{time}->{last_run};

                printf  OUTFILE "\n";

                write_tag ($p, 'puppet_noop_last_run',  $d->{time}->{last_run});
                write_tag ($p, 'puppet_noop_age',       $age);
                write_tag ($p, 'puppet_noop_runtime',   $d->{time}->{total});
                write_tag ($p, 'puppet_noop_changes',   $d->{changes}->{total});
                write_tag ($p, 'puppet_noop_success',   $d->{events}->{success});
                write_tag ($p, 'puppet_noop_failures',  $d->{events}->{failure});
        }
        else
        {
                printf  OUTFILE "\n";

                write_tag ($p, 'warning', "File '$fn_noop' not found for puppet staging run (noop).");
        }

        printf  OUTFILE "%s</puppet>\n", $prefix;
}

sub     read_puppet_conf
{
     my $fn     = shift;

        # A typical puppet configuration file looks like:
        #
        #       # Generated by Puppet (ofcourse)
        #       [agent]
        #       server = cppup03.prd.dsg-internal
        #       environment = production

     my ($server, $environment);

        if      (open (INFILE, "<$fn"))
        {
                while   (defined (my $line = <INFILE>))
                {
                        $line   =~ s/[\r\n]//g;

                        # Very simple parsing of key-value pairs, does not even look at section headers.

                        if      ($line =~ /^server[ \t]*=/)
                        {
                                $server = $line; $server =~ s/^server[ \t]*=[ \t]*//;
                        }
                        elsif   ($line =~ /^environment[ \t]*=/)
                        {
                                $environment = $line; $environment =~ s/^environment[ \t]*=[ \t]*//;
                        }
                }

                close   ($fn);
        }

        return  ($server, $environment);
}

sub     read_yaml
{
     my $fn     = shift;

     my %stack  = (); my $stack = \%stack; $stack->{0} = build_yaml_node();

     my $position = 0; # As YAML is coding by layout, keeping track of indentation is kind of important.
     my $line_nr  = 0;

     my $continue_on_next_line;

        open    (INFILE, "<$fn");

        while   (defined (my $line = <INFILE>))
        {
                $line =~ s/[\r\n]//g; $line =~ s/[ \t]*$//; $line_nr ++;

                if      ($line =~ /^---/)
                {
                        # Whoever made this up. Dunno what it means.
                }
                elsif   (($line =~ /^[ \t]*[#;]/) || ($line eq ''))
                {
                        # Skip comments and empty lines.
                }
                else
                {
                     my $trailer = $line; $line =~ s/^[ \s]*//;
                     my $indent  = length ($trailer) - length ($line);

                     my $node   = build_yaml_node();

#                       printf  "%5d %5d %5d %3d %3d | '%s'\n", $line_nr, $position, $indent, length ($trailer), length ($line), $line;

                        if      (defined ($continue_on_next_line))
                        {
                                if      ($indent <= $position)
                                {
                                        printf "Indentation on line %s (%s) is %s, should be more than %s\n", $line_nr, $line, $indent, $position;
                                        return undef;
                                }

                                $position = $indent;

                                $stack->{$position} = $continue_on_next_line;

                             my $type   = 'scalar';
                                $type   = 'array' if ($line =~ /^-/);
                                $type   = 'hash'  if ($line =~ /:/);

                                $continue_on_next_line = undef;
                        }
                        elsif   ($indent > $position)
                        {
                                printf "Indentation on line %s (%s) is %s, should be less or equal than %s\n", $line_nr, $line, $indent, $position;
                                return undef;
                        }
                        else
                        {
                                $position = $indent;
                        }

                     my $parent = $stack->{$position};

                        unless  (defined($parent))
                        {
                                printf "Indentation on line %s (%s) is %s, no parent found.\n", $line_nr, $line, $indent;
                                return undef;
                        }

                        push    (@{$parent->{content}}, $node);

                        $line   =~ s/^[ \t]*//;

                        if      ($line =~ /^-.*:/)
                        {
                                $node->{label} = $line; $node->{label} =~ s/[ \t]*:.*//;
                                $node->{value} = substr ($line, length ($node->{label})); $node->{value} =~ s/[ \t]*:[ \t]*//;
                                $node->{label} =~ s/^- *//;
                        }
                        elsif   ($line =~ /:$/)
                        {
                                $node->{label} = $line; $node->{label} =~ s/^- *//; $node->{label} =~ s/[ \t]*:$//;
                                $continue_on_next_line = $node;
                        }
                        elsif   ($line =~ /:/)
                        {
                                ($node->{label}, $node->{value}) = split (':', $line);

                                $node->{label} =~ s/[ \t]*$//;
                                $node->{value} =~ s/^[ \t]*//;
                        }
                        else
                        {
                                $node->{value} = $line; $node->{value} =~ s/^- *//;
                        }

                        if      (defined ($node->{value}) && ($node->{value} =~ /^"/))
                        {
                                # Handle a double-quoted string, which can be multiline, especially to please us.
                                # Should really do this for single quotes as well. And should handle arrays between square brackets.
                                # And who-knows-what-else, such as handling escape characters along the way.
                                # Reading the specs would be a great idea before writing a parser.
                                # So little time, so many things to do...

                                $node->{value} =~ s/^"//;

                                while   (($node->{value} !~ /"$/) && (defined (my $l2 = <INFILE>)))
                                {
                                        $l2 =~ s/[\r\n]//g; $l2 =~ s/^[ \s]*/ /; $l2 =~ s/[ \r]*$//;

                                        $node->{value} .= $l2;
                                }

                                $node->{value} =~ s/"$//;
                        }
                        elsif   (defined ($node->{value}) && ($node->{value} =~ /^'/))
                        {
                                $node->{value} =~ s/^'//;

                                while   (($node->{value} !~ /'$/) && (defined (my $l2 = <INFILE>)))
                                {
                                        $l2 =~ s/[\r\n]//g; $l2 =~ s/^[ \s]*/ /; $l2 =~ s/[ \r]*$//;

                                        $node->{value} .= $l2;
                                }

                                $node->{value} =~ s/'$//;
                        }
                }
        }

        close   (INFILE);

        return  $stack->{0};
}

sub     build_yaml_node
{
     my %node   = (); my $node = \%node;
     my @rows   = (); $node->{content} = \@rows;

        return  $node;
}

sub     flatten_node
{
     my $node   = shift;
     my $depth  = shift || 0;

     my $flat;

        if      ($depth > 10)
        {
                printf  "Recursion limit reached\n";

                return  undef;
        }

        if      (defined (my $child1 = $node->{content}->[0]))
        {
                if      (defined ($child1->{label}))
                {
                     my %n = (); $flat = \%n;
                }
                else
                {
                     my @n = (); $flat = \@n;
                }

             my $i;

                foreach my $child (@{$node->{content}})
                {
                        $i++;

                        if      (defined ($child1->{label}))
                        {
                             my $label  = $child->{label};

                                unless  (defined ($label))
                                {
                                        $label = sprintf "MISSING_LABEL_%10d", $i;
                                }

                                $flat->{$label} = flatten_node ($child, $depth + 1);
                        }
                        else
                        {
                                push (@{$flat}, flatten_node ($child, $depth + 1));
                        }
                }
        }
        elsif   (defined (my $v = $node->{value}))
        {
                return  $v;
        }

        return  $flat;
}
```

## Install zabbix repo
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/8/x86_64/zabbix-release-latest-5.0.el8.noarch.rpm
```
## install zabbix front-end 
```bash
dnf install zabbix-server-pgsql zabbix-web-pgsql zabbix-apache-conf zabbix-agent
```
## install postgresql 13
```bash
dnf module list postgresql
sudo dnf module enable postgresql:13
sudo dnf install postgresql-server
sudo postgresql-setup --initdb
sudo systemctl enable --now  postgresql
```
## Login to user postgres
```bash
su - postgres
psql
```
## Create zabbix database
```bash
CREATE DATABASE zabbix
    WITH OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'C'
    LC_CTYPE = 'C'
    TEMPLATE = template0;
```
## Create Zabbix user
```bash
CREATE USER zabbix WITH PASSWORD '<PASSWORD>';
```
## Grant privileges do zabbix user
```bash
GRANT ALL PRIVILEGES ON DATABASE zabbix TO zabbix;
```

## execute zcat with zabbix data
```bash
zcat /usr/share/doc/zabbix-server-pgsql/create.sql.gz | sudo -u zabbix psql zabbix
```
## Edit pg_hba file to allow MD5 login
```bash
vim /var/lib/pgsql/data/pg_hba.conf
local   all             all                                     md5
```
## Restart postgresql
```bash
systemctl restart postgresql && systemctl enable postgresql
```
## configure timezone in PHP
```bash
vim /etc/php-fpm.d/zabbix.conf
php_value[date.timezone] = America/Toronto
```
## add database access on zabbix server conf 
```bash
vim /etc/zabbix/zabbix_server.conf
```

start all services
```bash
systemctl enable --now zabbix-server zabbix-agent httpd php-fpm && tail -f /var/log/zabbix/zabbix_server.log
```




# Install zabbix proxy


## add repo
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/5.0/rhel/8/x86_64/zabbix-release-latest-5.0.el8.noarch.rpm
dnf clean all
```
## install zabbix
```bash
dnf install zabbix-proxy-pgsql
```
## install postgresql 13
```bash
dnf module list postgresql
sudo dnf module enable postgresql:16
sudo dnf install postgresql-server
sudo postgresql-setup --initdb
sudo systemctl enable --now  postgresql
```
## add user and database
```bash
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix_proxy
```
## insert zabbix data into database
```bash
zcat  /usr/share/doc/zabbix-proxy-pgsql/schema.sql.gz | sudo -u zabbix psql zabbix_proxy
```
## Edit pg_hba file to allow MD5 login
```bash
vim /var/lib/pgsql/data/pg_hba.conf
local   all             all                                     md5
```
## Restart postgresql
```bash
systemctl restart postgresql && systemctl enable postgresql
```
## edit zabbix_proxy configuration file and modify "Server=, Hostname=, database password"
```bash
vim /etc/zabbix/zabbix_proxy.conf
```
## Generate TLS PSKI for proxy 
```bash
openssl rand -hex 32 > /etc/zabbix/zabbix_proxy.psk
```
## add TLS PSKI configuration on zabbix file
```bash
TLSConnect=psk
TLSPSKFile=/etc/zabbix/zabbix_proxy.psk
TLSPSKIdentity=zabbix_proxy
```

# Process do dump and restore
```bash
pg_dump -U postgres -F c -b -v -f /tmp/zabbix_dump.dump zabbix

psql -U postgres -d zabbix -c "DROP SCHEMA public CASCADE;"
psql -U postgres -c "DROP DATABASE IF EXISTS zabbix;"
psql -U postgres -c "CREATE DATABASE zabbix;"

pg_restore -U postgres -d zabbix -v /tmp/zabbix_dump.dump
```

## DEFINITIVE DUMP AND RESTORE

#NOTES
backups located at > /mnt/backup/OnDisk/brppzbm01/pgbackup/zabbix <br>
copy to local storage > /data/pgsql/DUMP

#DUMP
```bash
pg_dump -U postgres -F c -b -v -f /tmp/zabbix_dump.dump zabbix
```

#RESTORE
```bash
DROP DATABASE zabbix;
CREATE DATABASE zabbix;
GRANT ALL PRIVILEGES ON DATABASE zabbix TO zabbix;
CREATE ROLE report;
pg_restore -U postgres -d zabbix --jobs=4 --verbose pg_dump.brppzbm01.prd.dsg-internal.5432.zabbix.20250112_220001.custom
```


cp -rfp /mnt/backup/OnDisk/brppzbm01/pgbackup/zabbix/pg_dump.brppzbm01.prd.dsg-internal.5432.zabbix.20250112_220001.custom .




