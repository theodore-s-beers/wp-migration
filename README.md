# Steps to deploy a WordPress site copy on a new server and domain

I'm writing this largely for myself, to document the process that I followed to clone a
WordPress site from one hosting provider to another. In particular, I'm getting ready to
move the website of [Middle East Medievalists](https://www.middleeastmedievalists.com/)
to new hosting, probably on a simple VPS, and I wanted to make sure it's feasible. So I
got a clone of the site up and running at <https://mem.t6e.dev/>.

1.  Back up site files

    - In my case, the original hosting provider offers a cPanel interface, which I used
      to request and download a full backup of site data. It came in a `.tar.gz` bundle,
      most of whose contents proved irrelevant.
    - What ended up being necessary were two things: the `wp-content` directory, and the
      `.sql` file of the main site database.
    - I could easily imagine more complex setups in which other directories/files would
      need to be backed up and transferred, but in this case, if I didn't have cPanel, I
      could have just used `rsync` or similar to transfer the key pieces.

2.  Set up new server

    - Provision a VPS (e.g., a DigitalOcean droplet)
    - Set up WordPress on the new server, using something pre-configured (or manually if
      necessary)
    - Point the new domain (e.g., `test.bar.dev`) to the server's IP address
    - Ensure there's a good SSL certificate via
      [Let's Encrypt](https://letsencrypt.org/)

3.  Transfer files

    - First, we'll transfer the `wp-content` directory taken from the original server to
      the new one, using e.g. `rsync` (always being careful with trailing slashes in the
      source and destination paths!).
    - We may want to either move or delete the default contents of `wp-content` on the
      new server, before placing the backed-up data there. e.g.,
      `mv /var/www/html/wp-content /var/www/html/wp-content-old`
    - Then we can do our transfer, e.g.,
      `rsync -avz /path/to/wp-content/ user@ip:/var/www/html/wp-content/`
    - The destination directory will (afaik) be created if it doesn't exist.
    - Second, we'll move the database backup (`.sql` file) to the new server. In this
      case, the destination location is arbitrary, since we'll just be using `mysql` to
      import the backup into a new database running on the new server. I placed the
      `.sql` file in the home directory on the server:
      `rsync -avz /path/to/backup.sql user@ip:~`

4.  Set file permissions

    - Ensure correct ownership for the transferred `wp-content` directory structure:
      `sudo chown -R www-data:www-data /var/www/html/wp-content`
    - If you used `rsync` in "archive mode" (with the `-a` flag), you should not need to
      fix any permissions. In fact, you may not even need to update the ownership—though
      I did. But do verify that `wp-content` and subsidiary directories have `755`
      permissions, and files have `644`.

5.  Move backup into new database

    - Log into MySQL on the server and create a new database (with a new database user,
      incl. credentials):

          ```sql
          CREATE DATABASE new_db;
          GRANT ALL PRIVILEGES ON new_db.\* TO 'dbuser'@'localhost' IDENTIFIED BY 'password';
          FLUSH PRIVILEGES;
          EXIT;
          ```

    - Import the database backup (you will be prompted for the password that you just
      set): `mysql -u dbuser -p new_db < /path/on/server/backup.sql`

6.  Update WordPress domain in database

    - Update URLs in the `wp_options` table (or similar; check the prefix!) to reflect
      the new domain:

          ```sql
          USE new_db;
          UPDATE wp_options SET option_value = 'https://test.bar.dev' WHERE option_name = 'siteurl';
          UPDATE wp_options SET option_value = 'https://test.bar.dev' WHERE option_name = 'home';
          ```

7.  Configure WordPress to use new database

    - Open the `wp-config.php` file on the new server:
      `sudo vim /var/www/html/wp-config.php`
    - Update database details to reflect the new database name, user, and password:

          ```php
          define( 'DB_NAME', 'new_db' );
          define( 'DB_USER', 'dbuser' );
          define( 'DB_PASSWORD', 'password' );
          define( 'DB_HOST', 'localhost' );
          ```

8.  Regenerate permalinks

    - Log into WordPress _in a browser_ at (e.g.) `test.bar.dev/wp-admin`
    - The admin credentials should be the same as on the original site (e.g., at
      `foo.com/wp-admin`)
    - Go to Settings > Permalinks and click Save Changes to regenerate `.htaccess` rules

9.  Verify and test site

    - Visit (e.g.) `test.bar.dev` and ensure the site is functioning properly
    - Test media, plugins, and content to verify that everything was transferred
    - Run a broken link checker and/or check for any hardcoded URLs still pointing to
      the old site URL (e.g., `foo.com`)

10. Clean up

    - Consider deleting files transferred during the migration process that are no
      longer necessary. In my case, this meant only the `.sql` file.
