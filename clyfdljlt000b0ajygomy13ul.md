---
title: "Installing MySQL on Ubuntu: A Beginner’s Guide"
seoTitle: "Installing MySQL on Ubuntu: A Beginner’s Guide"
seoDescription: "Beginner’s guide to installing MySQL on Ubuntu. Follow these steps to install, secure, start, and test your MySQL installation"
datePublished: Wed Jul 10 2024 05:04:38 GMT+0000 (Coordinated Universal Time)
cuid: clyfdljlt000b0ajygomy13ul
slug: installing-mysql-on-ubuntu-a-beginners-guide
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/Y9kOsyoWyaU/upload/27ca970b9f0cd9e06973a5f13d25f383.jpeg
tags: ubuntu, mysql, databases

---

MySQL is one of the most popular open-source relational database management systems in the world. Whether you’re setting up a web application, managing data for a project, or simply learning about databases, installing MySQL on your Ubuntu system is a crucial skill. In this step-by-step guide, we will walk you through the process of installing MySQL on an Ubuntu machine.

## **Step 1: Update Your Package Repository**

Before installing any software on Ubuntu, it’s essential to update the package repository to ensure you have the latest information about available packages. Open your terminal and run the following commands:

```bash
sudo apt update
```

The `sudo apt update` command is used to refresh the package information on a Debian-based Linux system. It ensures that your system’s package manager (APT, which stands for Advanced Package Tool) has the latest information about available packages and their versions from the configured software repositories.

Then Upgrade:

```bash
sudo apt upgrade
```

The `sudo apt upgrade` command is used to upgrade the installed packages on a Debian-based Linux system, such as Ubuntu. This command will search for updates to your installed packages and, if any updates are available, it will prompt you to confirm the upgrade. It is a good practice to regularly run this command to keep your system up to date with the latest software updates and security patches.

## **Step 2: Install MySQL**

Now, let’s install MySQL by using the following command:

```bash
sudo apt install mysql-server
```

This command will prompt you to confirm the installation by typing “Y” and then hitting Enter. The MySQL server package and dependencies will be downloaded and installed.

## **Step 3: Secure Your MySQL Installation**

After MySQL is installed, it’s important to secure it to prevent unauthorized access and enhance security. The following command will initiate the MySQL secure installation process:

```bash
sudo mysql_secure_installation
```

You will be asked a series of questions to configure security settings. It’s advisable to answer “Y” (yes) to most of the questions, but you can customize this to suit your requirements.

1. **Question:** Type your password and press `Y` to set up the `VALIDATE PASSWORD` component which checks whether the new password is strong enough. If you don’t want to `VALIDATE PASSWORD` Then press `N`to set (No).
    

1. 1. If you press yes then follow those steps:  Enter `0`, `1`, or `2` depending on the password strength you want to set :
        

* 1. * `0` – Low. The password consists of at least 8 characters.
            
        * `1` – Medium. The password consists of at least 8 characters (including numeric, mixed case, and special characters).
            
        * `2` – Strong. The password consists of at least 8 characters (including numeric, mixed case, and special characters, and compares the password to a dictionary file).
            
    2. Please set the password for root and enter the password you want to set.
        
    3. Once you specify the required strength, enter and re-enter the password.
        
    4. The program evaluates the strength of your password and requires confirmation `Y` to continue.
        

2. **Question:** Remove anonymous users.
    

**Answer:** To remove the demo/test user, press `Y`. If you want to keep them, press `N` to set it to ‘No’.

3. **Question:** Disallow root login remotely?
    

**Answer:** Press `Y` to disable remote access to the MySQL server. If you want to enable remote access, press `N`.

4. **Question:** Remove the test database and access to it.
    

**Answer:** `Y` to remove the test database and access to it or `N` for allow. This is a recommended security measure to remove the default test database, which is often unnecessary and can be a potential security risk if left in place. Removing it helps to enhance the security of your MySQL installation.

5. **Question:** Reload privilege tables?
    

**Answer:** `Y` for “Yes.” and `N` for “No”. Reloading the privilege tables is essential after making changes to the MySQL user accounts and permissions. This ensures that the changes take effect immediately and that MySQL enforces the updated user privileges. It’s necessary to properly secure and configure MySQL as per your settings during the installation process.

Your MySQL installation is now secure.

## **Step 4: Start and Enable MySQL**

MySQL may not start automatically after installation. To ensure MySQL starts with your system, you can use the following commands:

```bash
sudo systemctl start mysql
```

```bash
sudo systemctl enable mysql
```

Now, MySQL will start automatically every time your system boots.

## **Step 5: Check MySQL Status**

You can check if MySQL is running properly with the following command:

```bash
sudo systemctl status mysql
```

You should see a status message indicating that MySQL is active and running.

## **Step 6: Test Your MySQL Installation**

To verify that MySQL is functioning correctly, you can log in as the root user:

```bash
mysql -u root -p
```

You’ll be prompted to enter the root password you set during the secure installation. If you can log in without any issues, your MySQL installation is successful.

## **Conclusion**

Congratulations! You have successfully installed MySQL on your Ubuntu system. MySQL is a powerful and versatile database management system that can be used for various applications, from web development to data analysis. You can now start creating databases, and tables, and managing your data with MySQL on your Ubuntu machine.

**Happy Coding 😉**