# vagrant-centos7-docker


vagrant login
vagrant init nuaays/centos7-docker

vagrant up
vagrant halt
vagrant package ( vagrant package --output=centos7-docker )

mv package.box centos7-docker.box
vagrant box add --name centos7-docker centos7-docker.box
