# CPAN
http://search.cpan.org/~hikarine/BDB-Wrapper/

# Author
Hajime Kurita

## Technical web services by the author
### English
- https://ProgrammingLang.com/en/
- https://VpsRanking.com/en/

### Chinise
- https://ProgrammingLang.com/zh/
- https://VpsRanking.com/zh/

### Japanese
- https://ProgrammingLang.com/ja/
- https://VpsHikaku.com/
- https://RentalServerHikaku.com/
- https://SenyouServerHikaku.com/

# INSTALLATION
To install this module type the following:

   perl Makefile.PL
   make
   make test
   make install

DEPENDENCIES
BerkeleyDB.pm

# COPYRIGHT AND LICENCE
Copyright (C) 2008 by 1st class (http://www.1stclass.co.jp/)

This library is free software; you can redistribute it and/or modify
it under the same terms as Perl itself, either Perl version 5.8.8 or,
at your option, any later version of Perl 5 you may have available.

# Explanation
  BDB::Wrapper

  Wrapper module for BerkeleyDB.pm for easy usage of it.
  This will make it easy to use BerkeleyDB.pm.
  You can protect bdb file from the concurrent access and you can use BerkeleyDB.pm with less difficulty.
  This module is used on http://sakuhindb.com/ and is developed based on the requirement.

  Attention: If you use this module for the specified Berkeley DB file,
  please use this module for all access to the bdb.
  By it, you can control lock and strasaction of bdb files.
  BDB_HOMEs are created under /tmp/bdb_home in default option.

  Japanese: http://sakuhindb.com/pj/6_B4C9CDFDBFCDA4B5A4F3/13/list.html
  English: http://en.sakuhindb.com/pe/Administrator/19/list.html

#Example of basic usage
```
  #!/usr/bin/perl -w
  package test_bdb;
  use strict;
  use BDB::Wrapper;
  my $pro=new test_bdb;
  $pro->run();
  sub new(){
    my $self={};
    return bless $self;
  }

  sub run(){
    my $self=shift;
    $self->init_vars();
    $self->demo();
  }

  sub init_vars(){
    my $self=shift;
    $self->{'bdb'}='/tmp/test.bdb';
    $self->{'bdbw'}=new BDB::Wrapper;
  }

  sub demo(){
    my $self=shift;
    if(my $dbh=$self->{'bdbw'}->create_write_dbh($self->{'bdb'})){
      ###############
      # This is not must job but it will help to avoid unexpected result caused by unexpected process killing
      my $lock=$dbh->cds_lock();
      local $SIG{'INT'};
      local $SIG{'TERM'};
      local $SIG{'QUIT'};
      $SIG{'INT'}=$SIG{'TERM'}=$SIG{'QUIT'}=sub {$lock->cds_unlock();$dbh->db_close();};
      ###########
      if($dbh && $dbh->db_put('name', 'value')==0){
      }
      else{
        $lock->cds_unlock();
        $dbh->db_close() if $dbh;
        die 'Failed to put to '.$self->{'bdb'};
      }
      $lock->cds_unlock();
      $dbh->db_close() if $dbh;
    }

    if(my $dbh=$self->{'bdbw'}->create_read_dbh($self->{'bdb'})){
      my $value;
      if($dbh->db_get('name', $value)==0){
        print 'Name=name value='.$value."\n";
      }
      $dbh->db_close();
    }
  }
```

# Example of using transaction
```
  # Transaction Usage
  #!/usr/bin/perl -w
  package bdb_write;
  use strict;
  use BDB::Wrapper;

  my $pro = new bdb_write;
  $pro->run();
  sub new(){
    my $self={};
    return bless $self;
  }

  sub run(){
    my $self=shift;
    $self->{'bdbw'}=new BDB::Wrapper;
    # If you want to create bdb_home with transaction log under /home/txn_data/bdb_home/$BDBFILENAME/
    my ($dbh, $env)=$self->{'bdbw'}->create_write_dbh({'bdb'=>'/tmp/bdb_write.bdb', 'txn'=>1});
    my $txn = $env->txn_begin(undef, DB_TXN_NOWAIT);
  
    my $cnt=0;
    for(my $i=0;$i<1000;$i++){
      $dbh->db_put($i, $i*rand());
      $cnt=$i;
      if($cnt && $cnt%100==0){
        $txn->txn_commit();
        $txn = $env->txn_begin(undef, DB_TXN_NOWAIT);
      }
    }

    $txn->txn_commit();
    $env->txn_checkpoint(1,1,0);
    $dbh->db_close();
    chmod 0666, '/tmp/bdb_write.bdb';
    print "Content-type:text/html\n\n";
    print $cnt."\n";
  }
```

# methods
## new
  Creates an object of BDB::Wrapper
  
  If you set {'ram'=>1}, you can use /dev/shm/bdb_home for storing locking file for BDB instead of /tmp/bdbwrapper/bdb_home/.
  1 is default value.
  
  If you set {'no_lock'=>1}, the control of concurrent access will not be used. So the lock files are also not created.
  0 is default value.
  
  If you set {'cache'=>$CACHE_SIZE}, you can allocate cache memory of the specified bytes for using bdb files.
  The value can be overwritten by the cache value of create_write_dbh
  undef is default value.
  
  If you set {'wait'=>wait_seconds}, you can specify the seconds in which dead lock will be removed.
  22 is default value.
  
  If you set {'transaction'=>transaction_root_dir}, all dbh object will be created in transaction mode unless you don\'t specify transaction root dir in each method.
  0 is default value.

## create_env
  Creates Environment for BerkeleyDB
```
  create_env({'bdb'=>$bdb,
    'no_lock='>0(default) or 1,
    'cache'=>undef(default) or integer,
    'error_log_file'=>undef or $error_log_file,
    'transaction'=> 0==undef or 1 or $transaction_root_dir
    });
```

  no_lock and cache will overwrite the value specified in new but used only in this env
create_dbh
  Not recommened method. Please use create_read_dbh() or create_write_dbh().
  Creates database handler for BerkeleyDB
  This will be obsolete due to too much simplicity, so please don\'t use.
create_hash_ref
  Not recommended method. Please use create_write_dbh().
  Creates database handler for BerkeleyDB
  This will be obsolete due to too much simplicity, so please don\'t use.
create_write_dbh
  This returns database handler for writing or ($database_handler, $env) depeinding on the request.

```
  $self->create_write_dbh({'bdb'=>$bdb,
    'cache'=>undef(default) or integer,
    'hash'=>0 or 1,
    'dont_try'=>0 or 1,
    'no_lock'=>0(default) or 1,
    'sort_code_ref'=>$sort_code_reference,
    'sort' or 'sort_num'=>0 or 1,
    'transaction'=> 0==undef or $transaction_root_dir,
    'reverse_cmp'=>0 or 1,
    'reverse' or 'reverse_num'=>0 or 1
    });
```

  In the default mode, BDB file will be created as Btree;

  If you set 'hash' 1, Hash BDB will be created.

  If you set 'dont_try' 1, this module won\'t try to unlock BDB if it detects the situation in which deadlock may be occuring.

  If you set sort_code_ref some code reference, you can set subroutine for sorting for Btree.

  If you set sort or sort_num 1, you can use sub {$_[0] <=> $_[1]} for sort_code_ref.

  If you set reverse or reverse_num 1, you can use sub {$_[1] <=> $_[0]} for sort_code_ref.

  If you set reverse_cmp 1, you can use sub {$_[1] cmp $_[0]} for sort_code_ref.

  If you set transaction for storing transaction log, transaction will be used and ($bdb_handler, $transaction_handler) will be returned.

# create_read_dbh
  This returns database handler for reading or ($database_handler, $env) depeinding on the request.
```
  $self->create_read_dbh({
    'bdb'=>$bdb,
    'hash'=>0 or 1,
    'dont_try'=>0 or 1,
    'sort_code_ref'=>$sort_code_reference,
    'sort' or 'sort_num'=>0 or 1,
    'reverse_cmp'=>0 or 1,
    'reverse' or 'reverse_num'=>0 or 1,
    'transaction'=> 0==undef or 1 or $transaction_root_dir
    });
```

  In the default mode, BDB file will be created as Btree;

  If you set 'hash' 1, Hash BDB will be created.

  If you set 'dont_try' 1, this module won\'t try to unlock BDB if it detects the situation in which deadlock may be occuring.

  If you set sort_code_ref some code reference, you can set subroutine for sorting for Btree.

  If you set sort or sort_num 1, you can use sub {$_[0] <=> $_[1]} for sort_code_ref.

  If you set reverse or reverse_num 1, you can use sub {$_[1] <=> $_[0]} for sort_code_ref.

  If you set reverse_cmp 1, you can use sub {$_[1] cmp $_[0]} for sort_code_ref.

  If you set transaction 1, you will user /tmp/txn_data for the storage of transaction.
create_write_hash_ref
  Not recommended method. Please use create_write_dbh() instead of this method.
  This will creates hash for writing.
```
  $self->create_write_hash_ref({'bdb'=>$bdb,
    'hash'=>0 or 1,
    'dont_try'=>0 or 1,
    'sort_code_ref'=>$sort_code_reference,
    'sort' or 'sort_num'=>0 or 1,
    'reverse_cmp'=>0 or 1,
    'reverse' or 'reverse_num'=>0 or 1
    });
```
  In the default mode, BDB file will be created as Btree.

  If you set 'hash' 1, Hash BDB will be created.

  If you set 'dont_try' 1, this module won\'t try to unlock BDB if it detects the situation in which deadlock may be occuring.

  If you set sort_code_ref some code reference, you can set subroutine for sorting for Btree.

  If you set sort or sort_num 1, you can use sub {$_[0] <=> $_[1]} for sort_code_ref.

  If you set reverse or reverse_num 1, you can use sub {$_[1] <=> $_[0]} for sort_code_ref.

  If you set reverse_cmp 1, you can use sub {$_[1] cmp $_[0]} for sort_code_ref.
create_read_hash_ref
  Not recommended method. Please use create_read_dbh and cursor().
  This will creates database handler for reading.
```
  $self->create_read_hash_ref({
    'bdb'=>$bdb,
    'hash'=>0 or 1,
    'dont_try'=>0 or 1,
    'sort_code_ref'=>$sort_code_reference,
    'sort' or 'sort_num'=>0 or 1,
    'reverse_cmp'=>0 or 1,
    'reverse' or 'reverse_num'=>0 or 1
    });
```
  In the default mode, BDB file will be created as Btree.

  If you set 'hash' 1, Hash BDB will be created.

  If you set 'dont_try' 1, this module won\'t try to unlock BDB if it detects the situation in which deadlock may be occuring.

  If you set sort_code_ref some code reference, you can set subroutine for sorting for Btree.

  If you set sort or sort_num 1, you can use sub {$_[0] <=> $_[1]} for sort_code_ref.

  If you set reverse or reverse_num 1, you can use sub {$_[1] <=> $_[0]} for sort_code_ref.

  If you set reverse_cmp 1, you can use sub {$_[1] cmp $_[0]} for sort_code_ref.

  If you set use_env 1, you can use environment for this method.

##  rmkdir
  Code from CGI::Accessup.
  This creates the specified directory recursively.
```
  rmkdir($dir);
```

## get_bdb_home
  This will return bdb_home.
  You may need the information for recovery and so on.
```
  get bdb_home({
    'bdb'=>$bdb,
    'transaction'=>$transaction
    });
```
  OR
```
  get_bdb_home($bdb);
```

## clear_bdb_home
  This will clear bdb_home.
```
  clear_bdb_home({
    'bdb'=>$bdb,
    'transaction' => 0==undef or $transaction_root_dir
    });
```

  OR
```
  clear_bdb_home($bdb);
```

## record_error
  This will record error message to /tmp/bdb_error.log if you don\'t specify error_log_file
```
  record_error({
    'msg'=>$error_message,
    'error_log_file'=>$error_log_file
    });
```
  OR
```
  record_error($error_msg)
```