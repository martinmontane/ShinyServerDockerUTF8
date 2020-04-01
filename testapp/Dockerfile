FROM openanalytics/r-base
# Install Ubuntu packages
RUN apt-get update && apt-get install -y \
    sudo \
    pandoc \
    pandoc-citeproc \
    libcurl4-gnutls-dev \
    libcairo2-dev \
    libxt-dev \
    libssl-dev \
    libssh2-1-dev \
    software-properties-common \
    curl

RUN sudo add-apt-repository ppa:marutter/c2d4u \
         add-apt-repository ppa:marutter/c2d4u3.5

# Place to install libs from binary that might not be possible to install within R 
# RUN apt-get update && apt-get install -y 
# RUN sudo apt-get install r-cran-rcpp -y
  
# Install R packages that are required
# TODO: add further package if you need!
RUN R -e "install.packages(c('shiny'))"

# copy the app to the image
RUN mkdir /root/firstapp
COPY app /root/firstapp

COPY Rprofile.site /usr/lib/R/etc/

CMD ["R","-e", "shiny::runApp('/root/firstapp')"]

EXPOSE 3838
