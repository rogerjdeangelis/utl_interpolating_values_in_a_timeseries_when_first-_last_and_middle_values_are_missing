# utl_interpolating_values_in_a_timeseries_when_first-_last_and_middle_values_are_missing
Interpolating values in a timeseries when some first,last and middle values are missing.  Keywords: sas sql join merge big data analytics macros oracle teradata mysql sas communities stackoverflow statistics artificial inteligence AI Python R Java Javascript WPS Matlab SPSS Scala Perl C C# Excel MS Access JSON graphics maps NLP natural language processing machine learning igraph DOSUBL DOW loop stackoverflow SAS community.
    Interpolating values in a timeseries when some first,last and middle values are missing

    I carried forward and backward values to fill in missing mutiple first and last readings

    Recent update:
    Two improved solutions by Mark on end (much simpler)
    Keintz, Mark" <mkeintz@WHARTON.UPENN.EDU>
    
    By all means, do a double dow, but stop not only at last.id, but also when not missing(price).
    And then take advantage of a conditional lag queue update at those stopping points.
    
    At each stop in the first dow below:
    
    1 Get the lagged value of price (i.e. the most recent non missing price) as prior_price.
      But if the corresponding lagged value of ID does not match current id, then assign
      prior_price = current price.  This will accommodate carry backwards at the start of each id.
      
    2 Generate next_price=current price.  But if the do loop stop is due to last.id,
      and current price is missing then assign next_price=prior_price, which
      takes care of carrying forward at the end of an ID.
      
    s Reread the _N observations and get a weighted average
      of prior_price and next_price, with the weights proporitional
      to the distances from records containing prior_price and next_price .
      
    The trick” here is to conditionally generate lagged values of ID and PRICE.
    The lag (remember it’s a queue) is only updated when a valid price is in hand (or last.id=1).
    Previous Solutions
        Two Algorithms
             1. WPS/PROC R or IML/R (full solution)
             2. Not related but useful? Carry forward and carry backward(only if missing end) non missing values
    see
    https://stackoverflow.com/questions/48834869/filling-empty-values-in-sas
    INPUT
    =====
    SD1.HAVE total obs=25               |  RULES
                                        |
         DATE      PRODUCTID    PRICE   |  Price
                                        |
       1/1/2018        1           .    |  36  Carry backward
       1/2/2018        1           .    |  36
       1/3/2018        1           .    |  36
       1/4/2018        1          36    |  36
       1/5/2018        1           .    |  27 Interpolated
       1/6/2018        1          18    |  18 Carry Forward
       1/7/2018        1           .    |  18
       1/8/2018        1           .    |  18
       1/7/2018        2          50    |
       1/1/2018        2          13    |
       1/2/2018        2           .    |
       1/3/2018        2           .    |
       1/4/2018        2          17    |
       1/5/2018        2          18    |
       1/6/2018        2          19    |
       1/7/2018        2           .    |
    PROCESS (Two separate algorithms)
    =================================
      1.  WPS/PROC R or IML/R Working Code
          for (i in 1:3) {
            * select the group to work on;
            hav<-have[have$PRODUCTID==i,];
            * forward and backward fills;
            ends<-na.locf(na.locf(hav), fromLast = TRUE);
            * replace first and last only if missing;
            hav[1,]<-ends[1,];
            hav[nrow(hav),]<-ends[nrow(ends),];
            * interpolate between missing values;
            filmis<-as.data.frame(na.approx(hav$PRICE, 1:nrow(hav)));
            * append dataframes;
            want<-rbind(want,filmis);
         };
      2.  FORWARD and BACKGROUND Fills (all the code)
          data partial(keep=productid price finPrice);
            retain productid price;
            retain  been_there  val1st valLst finPrice location .; * not needed but I like to declare finPrice;
            do i=1 by 1 until (last.productid);
              set sd1.have;
              by productid;
              * get the first non missing and hold it;
              if price ne . and been_there=. then do;
                 been_there=1;
                 val1st=price;
                 put val1st=;
              end;
              * get the loaction of tha last non-missing and hold it;
              if price ne . then do;
                  valLst=price;
                  location=i;
              end;
            end;
            *fill in first last and intermediate;
            do i=1 by 1 until (last.productid);
              set sd1.have;
              by productid;
              if first.productid then finPrice=val1st;
              else if price ne . then finPrice=price;
              if i=location then finPrice=valLst;
              output;
            end;
            call missing(val1st, valLst, been_there, finPrice, location);
          run;quit;
    OUTPUT ( I like to keep date as number of days since 1/1/60)
    =============================================================
     1. WPS R IML/R
      WORK.WANT (R)
         DATE    PRODUCTID    PRICE     INTERP
        21185        1           .     36.0
        21186        1           .     36.0
        21187        1           .     36.0
        21188        1          36     36.0
        21189        1           .     27.0
        21190        1          18     18.0
        21191        1           .     18.0
        21192        1           .     18.0
        21191        2          50     50.0
        21185        2          13     13.0
        21186        2           .     14.3
        21187        2           .     15.7
        21188        2          17     17.0
        21189        2          18     18.0
        21190        2          19     19.0
        21191        2           .     19.0
        21185        3           .     24.0
        21186        3           .     24.0
        21187        3           .     24.0
        21188        3          24     24.0
        21189        3          25     25.0
        21190        3          26     26.0
        21191        3          27     27.0
        21193        3           .     27.0
        21194        3           .     27.0
     2. CARRY FORWARD BACKWARD
         WORK.PARTIAL total obs=25
        PRODUCTID    PRICE    FINPRICE
            1           .        36
            1           .        36
            1           .        36
            1          36        36
            1           .        36
            1          18        18
            1           .        18
            1           .        18
            2          50        50
            2          13        13
            2           .        13
            2           .        13
            2          17        17
            2          18        18
            2          19        19
            2           .        19
            3           .        24
            3           .        24
            3           .        24
            3          24        24
            3          25        25
            3          26        26
            3          27        27
            3           .        27
            3           .        27
    *                _              _       _
     _ __ ___   __ _| | _____    __| | __ _| |_ __ _
    | '_ ` _ \ / _` | |/ / _ \  / _` |/ _` | __/ _` |
    | | | | | | (_| |   <  __/ | (_| | (_| | || (_| |
    |_| |_| |_|\__,_|_|\_\___|  \__,_|\__,_|\__\__,_|
    ;
    options validvarname=upcase;
    libname sd1 "d:/sd1";
    data sd1.have(keep=productid date price);
       informat date mmddyy8.;
       input ProductID Date  Price;
    cards4;
    1 1/1/2018  .
    1 1/2/2018  .
    1 1/3/2018  .
    1 1/4/2018 36
    1 1/5/2018  .
    1 1/6/2018 18
    1 1/7/2018  .
    1 1/8/2018  .
    2 1/7/2018 50
    2 1/1/2018 13
    2 1/2/2018  .
    2 1/3/2018  .
    2 1/4/2018 17
    2 1/5/2018 18
    2 1/6/2018 19
    2 1/7/2018  .
    3 1/1/2018  .
    3 1/2/2018  .
    3 1/3/2018  .
    3 1/4/2018 24
    3 1/5/2018 25
    3 1/6/2018 26
    3 1/7/2018 27
    3 1/9/2018  .
    3 1/10/2018  .
    ;;;;
    run;quit;
    *                     ____
    __      ___ __  ___  |  _ \
    \ \ /\ / / '_ \/ __| | |_) |
     \ V  V /| |_) \__ \ |  _ <
      \_/\_/ | .__/|___/ |_| \_\
             |_|
    ;
    %utl_submit_wps64('
    libname sd1 "d:/sd1";
    options set=R_HOME "C:/Program Files/R/R-3.3.1";
    libname wrk "%sysfunc(pathname(work))";
    libname hlp "C:\Program_Files\SASHome\SASFoundation\9.4\core\sashelp";
    proc r;
    submit;
    source("C:/Program Files/R/R-3.3.1/etc/Rprofile.site", echo=T);
    library(haven);
    library(dplyr);
    library(zoo);
    have<-as.data.frame(read_sas("d:/sd1/have.sas7bdat"));
    want <- data.frame();
    for (i in 1:3) {
    hav<-have[have$PRODUCTID==i,];
    ends<-na.locf(na.locf(hav), fromLast = TRUE);
    hav[1,]<-ends[1,];
    hav[nrow(hav),]<-ends[nrow(ends),];
    filmis<-as.data.frame(na.approx(hav$PRICE, 1:nrow(hav)));
    want<-rbind(want,filmis);
    };
    colnames(want)<-"INTERP";
    endsubmit;
    import r=want data=wrk.wantwps;
    run;quit;
    ');
    data want;
      merge sd1.have wantwps;
    run;quit;
    * LOG;
    > have<-as.data.frame(read_sas("d:/sd1/have.sas7bdat"))
    > want <- data.frame()
    > for (i in 1:3) {hav<-have[have$PRODUCTID==i,]
    + ends<-na.locf(na.locf(hav), fromLast = TRUE)
    + hav[1,]<-ends[1,]
    + hav[nrow(hav),]<-ends[nrow(ends),]
    + filmis<-as.data.frame(na.approx(hav$PRICE, 1:nrow(hav)))
    + want<-rbind(want,filmis)
    + }
    > colnames(want)<-"INTERP"
    NOTE: Processing of R statements complete
    22        import r=want data=wrk.wantwps;
    NOTE: Creating data set 'WRK.wantwps' from R data frame 'want'
    NOTE: Data set "WRK.wantwps" has 25 observation(s) and 1 variable(s)
    23        run;
    NOTE: Procedure r step took :
          real time : 0.628
          cpu time  : 0.000
    *                  _                        __ _ _
     _   _ _ __     __| | _____      ___ __    / _(_) |
    | | | | '_ \   / _` |/ _ \ \ /\ / / '_ \  | |_| | |
    | |_| | |_) | | (_| | (_) \ V  V /| | | | |  _| | |
     \__,_| .__/   \__,_|\___/ \_/\_/ |_| |_| |_| |_|_|
          |_|
    ;
    %utl_submit_wps64('
    libname sd1 sas7bdat "d:/sd1";
    libname wrk sas7bdat "%sysfunc(pathname(work))";
    data wrk.partialwps(keep=productid price finPrice);
      retain productid price;
      retain  been_there  val1st valLst finPrice location .; * not needed but I like to declare finPrice;
      do i=1 by 1 until (last.productid);
        set sd1.have;
        by productid;
        * get the first non missing and hold it;
        if price ne . and been_there=. then do;
           been_there=1;
           val1st=price;
           put val1st=;
        end;
        * get the loaction of tha last non-missing and hold it;
        if price ne . then do;
            valLst=price;
            location=i;
        end;
      end;
      *fill in first last and intermediate;
      do i=1 by 1 until (last.productid);
        set sd1.have;
        by productid;
        if first.productid then finPrice=val1st;
        else if price ne . then finPrice=price;
        if i=location then finPrice=valLst;
        output;
      end;
      call missing(val1st, valLst, been_there, finPrice, location);
    run;quit;
    ');
    *__  __            _
    |  \/  | __ _ _ __| | __
    | |\/| |/ _` | '__| |/ /
    | |  | | (_| | |  |   <
    |_|  |_|\__,_|_|  |_|\_\
    ;
    Two improved solutions by Mark on end (much simpler)
    Keintz, Mark" <mkeintz@WHARTON.UPENN.EDU>

    By all means, do a double dow, but stop not only at last.id, but also when not missing(price).
    And then take advantage of a conditional lag queue update at those stopping points.


    At each stop in the first dow below:

    1 Get the lagged value of price (i.e. the most recent non missing price) as prior_price.
      But if the corresponding lagged value of ID does not match current id, then assign
      prior_price = current price.  This will accommodate carry backwards at the start of each id.

    2 Generate next_price=current price.  But if the do loop stop is due to last.id,
      and current price is missing then assign next_price=prior_price, which
      takes care of carrying forward at the end of an ID.

    s Reread the _N observations and get a weighted average
      of prior_price and next_price, with the weights proporitional
      to the distances from records containing prior_price and next_price .


    The trick” here is to conditionally generate lagged values of ID and PRICE.
    The lag (remember it’s a queue) is only updated when a valid price is in hand (or last.id=1).


    data want (drop=_:);
      if 0 then set have;
      do _N=1 by 1 until (last.id or not missing(price));
        set have;
        by id;
      end;
      _prior_price=ifn(id=lag(id),lag(price),price);
      _next_price=ifn(price=. & last.id,_prior_price,price);
      do _J=1 to _N;
        set have;
        if price=. then price= (1-(_J/_N))*_prior_price + (_J/_N)*_next_price;
        output;
      end;

    run;

    And instead of a classic double dow structure, you can have conditional single dows.
    I.e. read data outside of a do loop until a desired condition.
    The a execute a DO loop to reread the same observations, with an embedded SET and OUTPUT:

    data want (drop=_:);
      set have;
      by id;

      if last.id or not missing(price);
      _prior_price=ifn(id=lag(id),lag(price),price);
      _next_price=ifn(price=. & last.id,_prior_price,price);
      _N=coalesce(dif(_n_),_n_);
      do _J=1 to _N;
        set have;
        if price=. then price= (1-(_J/_N))*_prior_price + (_J/_N)*_next_price;
        output;
      end;

    run;




