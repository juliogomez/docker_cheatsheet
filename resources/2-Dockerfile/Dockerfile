FROM ubuntu
RUN apt-get update && apt-get install -y cowsay lolcat && rm -rf /var/lib/apt/lists/*
COPY myapp.sh /myapp/myapp.sh
CMD ["/myapp/myapp.sh"]
