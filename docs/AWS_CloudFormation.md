# AWS CloudFormation

## Contents

- [Resources](#Resources)
- [Summary](#Summary)
- [Cheatsheet](#Cheatsheet)

## Resources

- [resource](https://link)

## Summary

[source](https://link)

## Cheatsheet

### Convert JSON template to YAMl

This can be done on the console by viewing a template in [Designer](
https://eu-west-1.console.aws.amazon.com/cloudformation/designer) and selecting
the appropriate radio button for either JSON or YAML.

### Lists

The following list notation:

    DependsOn: [ RoleCodeBuild, RoleCodePipeline ]

is equivalent to:

    DependsOn:
    - RoleCodeBuild
    - RoleCodePipeline

### !Sub

#### Spread a long expression over multiple lines

(See: [ref.](
https://learnxinyminutes.com/docs/yaml)):

    literal_block: |
      This entire block of text will be the value of the 'literal_block' key,
      with line breaks being preserved.

      The literal continues until de-dented, and the leading indentation is
      stripped.

        Any lines that are 'more-indented' keep the rest of their indentation -
        these lines will be indented by 4 spaces.

    folded_style: >
      This entire block of text will be the value of 'folded_style', but this
      time, all newlines will be replaced with a single space.

      Blank lines, like above, are converted to a newline character.

        'More-indented' lines keep their newlines, too -
        this text will appear over two lines.

However, you can use join if you need no spaces or newline and still want to
spread the entry over multiple lines.

    DefinitionS3Location:
      Fn::Join:
      - ""
      - - !Sub "s3://${AWS::Region}-stack-${Environment}-private/"
        - "infrastructure/stack-templates/graphql/appsync-author.gql"
